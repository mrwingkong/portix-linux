# LXQt + KWin Wayland — Orientation-Aware Wallpaper (Tablet Mode)

**Persistent dual-wallpaper solution for landscape / portrait rotation on Artix (OpenRC) + LXQt + KWin.**

When you rotate into tablet/portrait mode, LXQt’s built-in wallpaper does not correctly re-layout. This setup replaces the LXQt wallpaper with `swaybg` and automatically switches between two properly sized images on every rotation while keeping desktop icons and the right-click menu.

---

## Features

- Automatic switch between landscape and portrait wallpapers
- Survives reboots and session restarts
- Does not interfere with LXQt desktop icons or context menu
- Works with KWin’s tablet mode / auto-rotation
- Explicitly enables the orientation sensor (`iio-sensor-proxy`)

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

Fold the screen — the display must rotate before the wallpaper script will switch images.

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

1. Right-click the desktop → **Desktop Preferences**
2. Background tab → select the SVG
3. Wallpaper mode: Stretch

---

## 3. Wallpaper paths

Edit these two lines in the script to match your actual files:

```
LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
```

---

## 4. The orientation script

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
# Orientation wallpaper switcher for LXQt + KWin Wayland

LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
MODE="fill"
OUTPUT="eDP-1"
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

    pkill -x swaybg 2>/dev/null
    sleep 0.4

    swaybg -o "$OUTPUT" -i "$img" -m "$MODE" &
    SWAYBG_PID=$!
}

# Wait for session to settle
sleep 5

current=$(get_orientation)
set_wallpaper "$current"

# Monitor for changes
while true; do
    sleep 1.0
    new=$(get_orientation)

    if [ "$new" != "$current" ]; then
        current="$new"
        sleep 1.5          # give KWin time to finish rotation
        set_wallpaper "$current"
    fi
done
EOF

chmod +x ~/.local/bin/lxqt-orientation-wallpaper.sh
```

---

## 5. Autostart

### Recommended – LXQt Session Settings

1. Open **LXQt Settings → Session Settings → Autostart**
2. Click **Add**
3. Name: `Orientation Wallpaper`
4. Command: `/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh`

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
# Sensor must be running
rc-service iio-sensor-proxy status

# Restart the wallpaper script
pkill -f lxqt-orientation-wallpaper
pkill -x swaybg
~/.local/bin/lxqt-orientation-wallpaper.sh &
```

Fold the screen and confirm the wallpaper switches.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Nothing happens when folding | Sensor not running / KWin not rotating | Check `iio-sensor-proxy` and System Settings → Display |
| Only black background | Desktop not transparent | Re-apply the transparent SVG |
| Wrong image | Incorrect path in script | Edit `LANDSCAPE` / `PORTRAIT` variables |
| Multiple swaybg processes | Old instances left running | `pkill -x swaybg` |

### Useful diagnostic commands

```bash
rc-service iio-sensor-proxy status
pgrep -a lxqt-orientation-wallpaper
pgrep -a swaybg

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
4. It restarts `swaybg` with the matching wallpaper.
5. PCManFM-Qt stays transparent so the wallpaper is visible and icons still work.

---

*Updated August 2026 – includes iio-sensor-proxy OpenRC service*
