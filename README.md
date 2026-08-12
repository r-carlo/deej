# Deej on Bazzite — Install Guide

Complete steps for building and running [Deej](https://github.com/omriharel/deej) (hardware volume mixer) on Bazzite, since it's not officially packaged for immutable Fedora-based distros.

**Hardware used:** Arduino clone with CH340 serial chip (`1a86:7523`)

## Prerequisites
- Arduino (or clone) with Deej firmware already flashed and sliders wired up
- Bazzite installed

## 1. Create a Fedora 39 Distrobox container

Fedora 40+ dropped `webkit2gtk-4.0`, which Deej's system tray dependency needs, so use Fedora 39:

```bash
distrobox create --name deej-fedora39 --image fedora:39
distrobox enter deej-fedora39
```

## 2. Install build dependencies (inside container)

```bash
sudo dnf install golang gtk3-devel libappindicator-gtk3-devel webkit2gtk4.0-devel git
```

> **If you use Pi-hole or another network-wide ad blocker:** whitelist `go.opencensus.io` before building. Otherwise `go build`/`go install` will fail with a `connection refused` error trying to fetch that dependency.

## 3. Clone and build Deej

```bash
git clone https://github.com/omriharel/deej.git
cd deej
go build -o deej ./pkg/deej/cmd/main.go

mkdir -p ~/go/bin
mv deej ~/go/bin/
chmod +x ~/go/bin/deej
```

## 4. Export the binary to the host

```bash
distrobox-export --bin ~/go/bin/deej
exit
```

This creates a wrapper script at `~/.local/bin/deej` on the host that transparently enters the container to run the real binary.

## 5. Configure Deej (on host)

```bash
cd ~/.local/bin
wget https://raw.githubusercontent.com/omriharel/deej/master/config.yaml
nano config.yaml
```

Key changes for Linux:
- `com_port`: change from `COM4` to your serial device, e.g. `/dev/ttyACM0`
- App names: drop `.exe` (e.g. `firefox`, `spotify`, `discord`)

Find your serial port:
```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

Keep `config.yaml` in the same directory as the `deej` wrapper (`~/.local/bin/`).

## 6. Fix serial port permissions (dialout group)

Bazzite is an atomic/immutable distro, so the `dialout` group isn't in `/etc/group` by default — you have to copy it over before `usermod` will work:

```bash
grep -E '^dialout:' /usr/lib/group | sudo tee -a /etc/group
sudo usermod -aG dialout $USER
sudo reboot
```

A simple logout isn't enough on Bazzite for this — reboot. Verify after rebooting:

```bash
groups | grep dialout
```

## 7. Test it manually

```bash
cd ~/.local/bin
./deej
```

Move the sliders and confirm volume changes. If this works but autostart doesn't (see below), that's expected — see the note on timing.

## 8. Autostart at login

Use a desktop autostart entry with a startup delay. The delay matters: Deej needs the distrobox container **and** the audio system (PipeWire/PulseAudio) to be fully up before it connects, otherwise it detects the Arduino but silently stops controlling volume after about 10 seconds.

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/deej.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Deej Volume Mixer
Exec=/bin/bash -c "sleep 20 && cd ~/.local/bin && ./deej"
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Comment=Hardware volume mixer
EOF

chmod +x ~/.config/autostart/deej.desktop
```

Reboot to test. Wait ~25 seconds after login before checking the sliders.

> A `systemd --user` service was also tried for this but wasn't necessary in the end — the plain desktop autostart entry with a delay was simpler and worked reliably.

## Troubleshooting

**Build fails with `go.opencensus.io` connection refused**
Pi-hole (or similar) is blocking the domain. Whitelist it and rebuild.

**`webkit2gtk4.0-devel` not found**
You're likely on Fedora 40+ inside the container. Recreate the container with `--image fedora:39`.

**Arduino not detected / permission denied on serial port**
Confirm `dialout` group membership (Step 6) and that you've rebooted since adding it.

**Sliders stop responding ~10 seconds after login, but work fine when run manually**
This is a startup timing issue — the audio system or container isn't ready yet when Deej connects. Increase the `sleep` delay in the autostart entry (Step 8) if 20 seconds isn't enough on your machine.

**`distrobox-export --bin` fails with "cannot find" the binary**
The build didn't actually succeed — check for errors in `go build` output before trying to export.
