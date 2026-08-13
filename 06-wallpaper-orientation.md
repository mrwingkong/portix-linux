# 06 – Wallpaper Orientation (swaybg)

Orientation-aware wallpapers for tablet mode (landscape ↔ portrait) using swaybg.

When the device rotates, LXQt’s built-in wallpaper does not re-layout correctly. This setup replaces it with swaybg and automatically switches between two properly sized images.

---

## Prerequisites (already done in 01 / 02)

These are normally required and were installed in the earlier stages:

- Packages: `swaybg`, `iio-sensor-proxy`, `python-pyqt6`
- `~/.local/bin` exists and is in `$PATH`

If any are missing:

```bash
sudo pacman -S --needed swaybg iio-sensor-proxy python-pyqt6
mkdir -p ~/.local/bin
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 1. Enable the orientation sensor (critical)

Without this, KWin will not rotate when you fold the screen.

### Create OpenRC service

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
```

```bash
sudo chmod +x /etc/init.d/iio-sensor-proxy
sudo rc-update add iio-sensor-proxy default
sudo rc-service iio-sensor-proxy start
```

Verify:

```bash
rc-service iio-sensor-proxy status
```

Fold the screen – the display must rotate before the wallpaper script will switch images.

---

## 2. Make the LXQt desktop transparent

PCManFM-Qt must be transparent so swaybg can draw underneath it.

```bash
mkdir -p ~/Pictures
```

```bash
cat > ~/Pictures/transparent.svg << 'EOF'
<svg width="200" height="200" version="1.1"
     xmlns="http://www.w3.org/2000/svg"
     viewBox="0 0 200 200"/>
EOF
```

```bash
pcmanfm-qt --set-wallpaper ~/Pictures/transparent.svg --wallpaper-mode stretch
```

Or do it graphically:

1. Right-click the desktop → **Desktop Preferences**
2. Background tab → select the SVG
3. Wallpaper mode: Stretch

---

## 3. Wallpaper paths

Edit the two variables in the script below to match your actual image files.

Suggested locations:

```
LANDSCAPE="/home/myname/Pictures/landscape.png"
PORTRAIT="/home/myname/Pictures/portrait.png"
```

---

## 4. Orientation script

```bash
cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
# Orientation wallpaper switcher for LXQt + KWin

LANDSCAPE="/home/myname/Pictures/landscape.png"
PORTRAIT="/home/myname/Pictures/portrait.png"
OUTPUT="eDP-1"
MODE="fill"

get_orientation() {
    python3 -c '
from PyQt6.QtGui import QGuiApplication
import sys
app = QGuiApplication(sys.argv)
s = app.primaryScreen()
w, h = s.size().width(), s.size().height()
print("portrait" if h > w else "landscape")
' 2>/dev/null || echo "landscape"
}

set_wallpaper() {
    local orient="$1"
    local img
    if [ "$orient" = "portrait" ]; then
        img="$PORTRAIT"
    else
        img="$LANDSCAPE"
    fi

    [ -f "$img" ] || return

    pkill -x swaybg 2>/dev/null
    sleep 0.4
    swaybg -o "$OUTPUT" -i "$img" -m "$MODE" &
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
4. Command: `/home/myname/.local/bin/lxqt-orientation-wallpaper.sh`

### Alternative – desktop file

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/lxqt-orientation-wallpaper.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Orientation Wallpaper
Exec=/home/myname/.local/bin/lxqt-orientation-wallpaper.sh
X-GNOME-Autostart-enabled=true
OnlyShowIn=LXQt;
EOF
```

---

## 6. Test right now

```bash
rc-service iio-sensor-proxy status
pkill -f lxqt-orientation-wallpaper 2>/dev/null
pkill -x swaybg 2>/dev/null
~/.local/bin/lxqt-orientation-wallpaper.sh &
```

Fold the screen and confirm the wallpaper switches.

---

## 7. Troubleshooting

| Symptom                        | Likely cause                  | Fix                                      |
|--------------------------------|-------------------------------|------------------------------------------|
| Nothing happens when folding   | Sensor not running / KWin not rotating | Check `iio-sensor-proxy` and Display settings |
| Only black background          | Desktop not transparent       | Re-apply the transparent SVG             |
| Wrong image                    | Incorrect path in script      | Edit `LANDSCAPE` / `PORTRAIT` variables  |
| Multiple swaybg processes      | Old instances left running    | `pkill -x swaybg`                        |

Useful diagnostics:

```bash
rc-service iio-sensor-proxy status
pgrep -a lxqt-orientation-wallpaper
pgrep -a swaybg
```

```bash
python3 -c '
from PyQt6.QtGui import QGuiApplication
import sys
app = QGuiApplication(sys.argv)
s = app.primaryScreen()
print(s.size().width(), "x", s.size().height())
'
```

---

*Artix OpenRC only – no systemd.*
