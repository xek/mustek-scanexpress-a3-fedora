# Mustek ScanExpress A3 USB 1200 PRO on Fedora

The scanner (USB ID `055f:040b`) is not supported by any current SANE backend on Fedora.
This setup runs the old SANE 1.0.19 driver (`mustek_usb2`) inside an Ubuntu 9.04 Jaunty i386
container, exposing it to the host via the SANE network protocol so that `simple-scan` and
`scanimage` work transparently.

**Requires an x86_64 host.** The container image is 32-bit i386 and runs natively on x86_64
without emulation. Other architectures (ARM, etc.) are not supported.

Tested on Fedora 43.

## Files

- `Dockerfile.saned` — the container image
- `libsane_1.0.19-1_i386.deb` — custom-built libsane with the A3 USB 1200 PRO driver

## One-time host setup (requires root)

These steps only need to be done once per machine.

### 1. Allow containers to access USB devices (SELinux)

```bash
sudo setsebool -P container_use_devices=true
```

### 2. udev rule — give your user read/write access to the scanner

```bash
echo "SUBSYSTEM==\"usb\", ATTRS{idVendor}==\"055f\", ATTRS{idProduct}==\"040b\", MODE=\"0664\", GROUP=\"$USERNAME\"" \
  | sudo tee /etc/udev/rules.d/70-mustek-a3.rules

sudo udevadm control --reload-rules
sudo udevadm trigger --attr-match=idVendor=055f --attr-match=idProduct=040b
```

Verify with `lsusb -d 055f:040b` to find the bus/device numbers, then `ls -la /dev/bus/usb/BUS/DEVICE` should show your group as owner.

### 3. Configure the SANE net backend on the host

```bash
echo 'localhost' | sudo tee -a /etc/sane.d/net.conf
```

## Build the image

Clone or download this repository, then from the repo directory:

```bash
podman build -f Dockerfile.saned -t mustek-saned .
```

## Run the container

Find the scanner's USB device node first (bus/device numbers can change after replug):

```bash
lsusb -d 055f:040b
# Example output: Bus 003 Device 044: ID 055f:040b Mustek Systems, Inc. ScanExpress A3 USB 1200 PRO
```

Start the container, substituting the correct bus and device numbers:

```bash
podman run -d \
  --device /dev/bus/usb/BUS/DEVICE \
  --network=host \
  --name scanner \
  mustek-saned
```

`--network=host` is required because the SANE network protocol opens a second random-port
socket for image data transfer in addition to the control port (6566). Without host networking
only the control port is reachable and scans fail silently.

Verify the scanner is visible:

```bash
scanimage -L
# Should show: net:localhost:mustek_usb2:libusb:BUS:DEVICE
```

## After replug

If you unplug and replug the scanner, the device node number may change. Stop and restart
the container with the updated node from `lsusb -d 055f:040b`:

```bash
podman rm -f scanner
podman run -d --device /dev/bus/usb/BUS/DEVICE --network=host --name scanner mustek-saned
```

## Stop the container

```bash
podman rm -f scanner
```

## Design notes

- The container is Ubuntu Jaunty i386 (`lpenz/ubuntu-jaunty-i386`) — needed for the old 32-bit
  SANE 1.0.19 library that supports this scanner.
- `saned` is managed by a Python inetd wrapper that handles one connection at a time
  (sequential), ensuring the USB device is fully released between scans.
- `saned` is killed after 120 seconds if it doesn't exit cleanly — this handles the
  protocol-version hang between saned 1.0.3 (server) and sane-backends 1.4.x (client).
- The `A3IIIU2 Spicall:` debug messages from the scanner firmware go to stderr, which is
  redirected to `/dev/null`, so they never appear in scan output.
- The `mustek_usb2` backend detects the scanner as "ScanExpress A3 USB 600 Pro" — this is
  normal; the driver supports both the 600 and 1200 DPI variants under the same name.
