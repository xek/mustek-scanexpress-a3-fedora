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

### 2. udev rule — give your user access to the scanner and create a stable device symlink

```bash
echo "SUBSYSTEM==\"usb\", ATTRS{idVendor}==\"055f\", ATTRS{idProduct}==\"040b\", MODE=\"0664\", GROUP=\"$USERNAME\", SYMLINK+=\"mustek-a3\"" \
  | sudo tee /etc/udev/rules.d/70-mustek-a3.rules

sudo udevadm control --reload-rules
sudo udevadm trigger --attr-match=idVendor=055f --attr-match=idProduct=040b
```

Verify: `ls -la /dev/mustek-a3` should exist, and `ls -la $(readlink -f /dev/mustek-a3)` should show your group as owner.

### 3. Configure the SANE net backend on the host

```bash
echo 'localhost' | sudo tee -a /etc/sane.d/net.conf
```

## Build the image

Clone or download this repository, then from the repo directory:

```bash
podman build -f Dockerfile.saned -t mustek-saned .
```

## Run at boot (recommended)

Create a systemd user service so the container starts automatically:

```bash
mkdir -p ~/.config/systemd/user
cat > ~/.config/systemd/user/mustek-scanner.service << 'EOF'
[Unit]
Description=Mustek ScanExpress A3 USB 1200 PRO SANE container
After=default.target

[Service]
ExecStart=podman run --rm --device /dev/mustek-a3 --network=host --name scanner mustek-saned
ExecStop=podman stop scanner
Restart=on-failure
RestartSec=5

[Install]
WantedBy=default.target
EOF

systemctl --user daemon-reload
systemctl --user enable --now mustek-scanner.service
loginctl enable-linger $USER
```

`loginctl enable-linger` makes user services start at boot even without an active login session.

Verify the scanner is visible:

```bash
scanimage -L
# Should show: net:localhost:mustek_usb2:libusb:...
```

## Run manually (one-off)

```bash
podman run -d \
  --device /dev/mustek-a3 \
  --network=host \
  --name scanner \
  mustek-saned
```

## Stop

```bash
systemctl --user stop mustek-scanner.service   # if using the service
# or
podman rm -f scanner
```

## Design notes

- The container is Ubuntu Jaunty i386 (`lpenz/ubuntu-jaunty-i386`) — needed for the old 32-bit
  SANE 1.0.19 library that supports this scanner.
- The udev symlink `/dev/mustek-a3` provides a stable device path that survives replug and
  reboots, so the service doesn't need updating if the USB bus/device numbers change.
- `saned` is managed by a Python inetd wrapper that handles one connection at a time
  (sequential), ensuring the USB device is fully released between scans.
- `saned` is killed after 120 seconds if it doesn't exit cleanly — this handles the
  protocol-version hang between saned 1.0.3 (server) and sane-backends 1.4.x (client).
- `--network=host` is required because the SANE network protocol opens a second random-port
  socket for image data transfer. Without it, only the control port (6566) is reachable and
  scans fail silently.
- The `A3IIIU2 Spicall:` debug messages from the scanner firmware go to stderr, which is
  redirected to `/dev/null`, so they never appear in scan output.
- The `mustek_usb2` backend detects the scanner as "ScanExpress A3 USB 600 Pro" — this is
  normal; the driver supports both the 600 and 1200 DPI variants under the same name.
