# LXQt + KWin Wayland — Orientation-Aware Wallpaper (Tablet Mode)

**Robust dual-wallpaper solution for landscape / portrait rotation on Artix (OpenRC) + LXQt + KWin.**

This version includes the missing pieces that usually cause the portrait wallpaper to vanish or never switch.

---

## Features

- Automatic switch between landscape and portrait wallpapers
- Survives reboots and session restarts
- Keeps desktop icons and right-click menu
- Works with KWin tablet mode / auto-rotation
- Explicitly enables the orientation sensor (`iio-sensor-proxy`)
- More reliable restart of `swaybg` after rotation

---

## 0. Enable the orientation sensor (critical)

Without this, KWin will **not** rotate when you fold the screen.

### Install

```bash
sudo pacman -S iio-sensor-proxy
```

### Create OpenRC service (Artix has none by default)

```bash
sudo tee /etc/init.d/iio-sensor-proxy > /dev/null << 'EOF'
#!/sbin/openrc-run

name="iio-sensor-proxy"
description="IIO Sensor Proxy"
command="/usr/lib/iio-sensor-proxy"
command_background=true
pidfile="/run/${RC_SVCNAME}.pid"

depend() {
    need dbus
}
EOF

sudo chmod +x /etc/init.d/iio-sensor-proxy
sudo rc-update add iio-sensor-proxy default
sudo rc-service iio-sensor-proxy start
```

### Verify

```bash
rc-service iio-sensor-proxy status
```

Fold the screen — the display **must** rotate before anything else will work.

---

## 1. Required packages

```bash
sudo pacman -S --needed swaybg python-pyqt6
```

---

## 2. Make the LXQt desktop transparent

PCManFM-Qt must be transparent so `swaybg` can draw underneath it.

```bash
cat > ~/Pictures/transparent.svg << 'EOF'
<svg width="200" height="200" version="1.1"
     xmlns="http://www.w3.org/2000/svg"
     viewBox="0 0 200 200"/>
EOF

pcmanfm-qt --set-wallpaper ~/Pictures/transparent.svg --wallpaper-mode stretch
```

Or do it graphically:

1. Right-click desktop → **Desktop Preferences**
2. Background tab → select the SVG
3. Mode: Stretch

---

## 3. Wallpaper paths

Edit these two lines in the script to match your files:

```
LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
```

---

## 4. The orientation script (improved)

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
# Robust orientation wallpaper switcher for LXQt + KWin Wayland

LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
MODE="fill"
OUTPUT="eDP-1"          # change if kscreen-doctor -o shows a different name
SWAYBG_PID=""

get_orientation() {
    dims=$(python3 -c '
from PyQt6.QtGui import QGuiApplication
import sys
app = QGuiApplication(sys.argv)
s = app.primaryScreen()
print(s.size().width(), s.size().height())
' 2>/dev/null)

    w=${dims%% *}
    h=${dims##* }
    if [ -z "$w" ] || [ -z "$h" ]; then
        echo "landscape"
        return
    fi
    if [ "$w" -gt "$h" ]; then
        echo "landscape"
    else
        echo "portrait"
    fi
}

set_wallpaper() {
    local orient="$1"
    local img

    if [ "$orient" = "landscape" ]; then
        img="$LANDSCAPE"
    else
        img="$PORTRAIT"
    fi

    # Clean kill
    pkill -x swaybg 2>/dev/null
    sleep 0.5

    # Force the correct output and give KWin time
    swaybg -o "$OUTPUT" -i "$img" -m "$MODE" &
    SWAYBG_PID=$!
}

# Wait for session to settle
sleep 6

current=$(get_orientation)
set_wallpaper "$current"

# Monitor for changes
while true; do
    sleep 1.0
    new=$(get_orientation)

    if [ "$new" != "$current" ]; then
        current="$new"
        # Important: wait for KWin to finish the rotation geometry change
        sleep 1.8
        set_wallpaper "$current"
    fi
done
EOF

chmod +x ~/.local/bin/lxqt-orientation-wallpaper.sh
```

---

## 5. Autostart

### Recommended – LXQt Session Settings

1. **LXQt Settings → Session Settings → Autostart**
2. Add → Name: `Orientation Wallpaper`
3. Command: `/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh`

### Alternative – desktop file

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/lxqt-orientation-wallpaper.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Orientation Wallpaper
Exec=/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh
X-GNOME-Autostart-enabled=true
OnlyShowIn=LXQt;
EOF
```

---

## 6. Test right now

```bash
# Make sure sensor is running
rc-service iio-sensor-proxy status

# Kill any old instance
pkill -f lxqt-orientation-wallpaper
pkill -x swaybg

# Start fresh
~/.local/bin/lxqt-orientation-wallpaper.sh &
```

Then fold the screen and watch.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Nothing happens when folding | Sensor not running / KWin not rotating | Check `iio-sensor-proxy` and System Settings → Display |
| Portrait wallpaper appears then vanishes | Timing / swaybg race | The new script has longer delays – use it |
| Only black background | Desktop not transparent | Re-apply the transparent SVG |
| Wrong output | Multiple monitors / wrong name | Change `OUTPUT="eDP-1"` after checking `kscreen-doctor -o` |

### Useful diagnostic commands

```bash
# Is the sensor alive?
rc-service iio-sensor-proxy status

# Is the script running?
pgrep -a lxqt-orientation-wallpaper

# Is swaybg running?
pgrep -a swaybg

# Current logical size (run while folded and while open)
python3 -c '
from PyQt6.QtGui import QGuiApplication
import sys
app = QGuiApplication(sys.argv)
s = app.primaryScreen()
print(s.size().width(), "x", s.size().height())
'
```

---

## How it works

1. `iio-sensor-proxy` feeds orientation data to KWin.
2. KWin rotates the output → logical width/height changes.
3. The script detects the change via Qt.
4. It waits for the geometry change to settle, then restarts `swaybg` on the correct output.
5. PCManFM-Qt stays transparent so the wallpaper is visible and icons still work.

---

*Updated August 2026 – includes sensor service + more robust rotation handling*
