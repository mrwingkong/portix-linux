# Tablet Mode Setup for Artix Linux + OpenRC + LXQt + labwc

This guide documents a working tablet-mode setup for a 2-in-1 convertible (tested on Aura X1 / ThinkPad-style device with Intel Core Ultra) running:

- Artix Linux (OpenRC)
- LXQt
- labwc (Wayland)
- Touchscreen + accelerometer + tablet-mode switch

Features achieved:
- Automatic screen rotation based on orientation
- Touchscreen calibration follows rotation
- Physical keyboard + touchpad automatically disabled in tablet mode
- On-screen keyboard (wvkbd) appears in tablet mode
- On-screen keyboard disappears in laptop mode

---

## 1. Install required packages

```bash
sudo pacman -S iio-sensor-proxy wlr-randr libinput libinput-tools
yay -S wvkbd
```

---

## 2. Create OpenRC service for iio-sensor-proxy

```bash
sudo tee /etc/init.d/iio-sensor-proxy > /dev/null << 'EOF'
#!/sbin/openrc-run

command="/usr/lib/iio-sensor-proxy"
command_background=yes
pidfile="/run/iio-sensor-proxy.pid"

depend() {
    need dbus
}
EOF

sudo chmod +x /etc/init.d/iio-sensor-proxy
sudo rc-update add iio-sensor-proxy default
sudo rc-service iio-sensor-proxy start
```

Test sensors:

```bash
monitor-sensor
```

---

## 3. Auto-rotation script

Create the script:

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/2in1screen << 'EOF'
#!/bin/bash
OUTPUT="eDP-1"          # Change if your internal display has a different name
RC="$HOME/.config/labwc/rc.xml"

# Ensure calibrationMatrix exists
if [ ! -f "$RC" ]; then
    mkdir -p "$(dirname "$RC")"
    cat > "$RC" << 'EORC'
<?xml version="1.0"?>
<labwc_config>
  <libinput>
    <device>
      <calibrationMatrix>1 0 0 0 1 0</calibrationMatrix>
    </device>
  </libinput>
</labwc_config>
EORC
elif ! grep -q calibrationMatrix "$RC"; then
    sed -i '/<\/labwc_config>/i\  <libinput>\n    <device>\n      <calibrationMatrix>1 0 0 0 1 0</calibrationMatrix>\n    </device>\n  </libinput>' "$RC"
fi

monitor-sensor | while read -r line; do
    case "$line" in
        *normal*)
            wlr-randr --output "$OUTPUT" --transform normal
            sed -i 's|<calibrationMatrix>.*</calibrationMatrix>|<calibrationMatrix>1 0 0 0 1 0</calibrationMatrix>|' "$RC"
            labwc -r
            ;;
        *left-up*)
            wlr-randr --output "$OUTPUT" --transform 90
            sed -i 's|<calibrationMatrix>.*</calibrationMatrix>|<calibrationMatrix>0 -1 1 1 0 0</calibrationMatrix>|' "$RC"
            labwc -r
            ;;
        *right-up*)
            wlr-randr --output "$OUTPUT" --transform 270
            sed -i 's|<calibrationMatrix>.*</calibrationMatrix>|<calibrationMatrix>0 1 0 -1 0 1</calibrationMatrix>|' "$RC"
            labwc -r
            ;;
        *bottom-up*)
            wlr-randr --output "$OUTPUT" --transform 180
            sed -i 's|<calibrationMatrix>.*</calibrationMatrix>|<calibrationMatrix>-1 0 1 0 -1 1</calibrationMatrix>|' "$RC"
            labwc -r
            ;;
    esac
done
EOF

chmod +x ~/.local/bin/2in1screen
```

---

## 4. Tablet-mode + On-screen keyboard script

Because the current session may not yet have the `input` group active, we use a small passwordless sudo rule.

### 4.1 Allow passwordless sudo for the detection tools

```bash
sudo EDITOR=nano visudo
```

Add this line at the bottom:

```
YOUR_USERNAME ALL=(ALL) NOPASSWD: /usr/bin/stdbuf, /usr/bin/libinput
```

(Replace `YOUR_USERNAME` with your actual username.)

### 4.2 Create the tablet-mode script

```bash
cat > ~/.local/bin/tablet-mode << 'EOF'
#!/bin/bash

OSK="wvkbd-mobintl"

# Portrait height (-H) and Landscape height (-L)
# Adjust these numbers to your preference
OPTS="-H 280 -L 220"

sudo stdbuf -oL libinput debug-events | while read -r line; do
    if echo "$line" | grep -q "tablet-mode state 1"; then
        pkill -x "$OSK"
        sleep 0.3
        $OSK $OPTS &
    elif echo "$line" | grep -q "tablet-mode state 0"; then
        pkill -x "$OSK"
    fi
done
EOF

chmod +x ~/.local/bin/tablet-mode
```

---

## 5. Autostart both scripts with labwc

```bash
mkdir -p ~/.config/labwc
nano ~/.config/labwc/autostart
```

Add these lines:

```bash
~/.local/bin/2in1screen &
~/.local/bin/tablet-mode &
```

Reload labwc:

```bash
labwc -r
```

---

## 6. Optional: Add user to input group (recommended)

```bash
sudo usermod -aG input $USER
```

After a full reboot the `input` group will be active. You can then remove the sudoers rule and switch the tablet-mode script to a non-sudo version if desired.

---

## 7. Testing

- Rotate the device → screen and touch should follow orientation.
- Fold fully into tablet mode → physical keyboard disabled, on-screen keyboard appears.
- Unfold → on-screen keyboard disappears, physical keyboard works again.

---

## Notes / Tweaks

- Change the internal display name if needed (`wlr-randr` to check).
- Adjust `-H` (portrait) and `-L` (landscape) heights in the tablet-mode script.
- `wvkbd-deskintl` can be used instead of `wvkbd-mobintl` if preferred.
- labwc has no animations, so rotation is instant.

---

## Credits

Setup developed and tested on Artix Linux + OpenRC + LXQt + labwc with a 2-in-1 convertible (Aura X1 / similar ThinkPad-style hardware).
```

The Markdown file has been created at:

**`/home/workdir/artifacts/tablet-mode-setup-artix-labwc.md`**

You can download it from the artifacts folder or copy it to your GitHub repository.
