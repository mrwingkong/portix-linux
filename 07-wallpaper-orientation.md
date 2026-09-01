# 07 – Wallpaper Orientation (swaybg)

Orientation-aware wallpapers for tablet mode (landscape ↔ portrait) using swaybg.

Also recovers the LXQt panel and desktop wallpaper when a display is plugged in or unplugged.

When the device rotates, LXQt’s built-in wallpaper does not re-layout correctly. This setup replaces it with swaybg and automatically switches between two images. One swaybg instance paints **every** connected screen. You do not add display names when you plug in a monitor.

---

## Prerequisites (already done in 01 / 02)

These are normally required and were installed in the earlier stages:

- Packages: `swaybg`, `iio-sensor-proxy`, `python-pyqt6`, `kscreen`
- `~/.local/bin` exists and is in `$PATH`

If any are missing:

```bash
sudo pacman -S --needed swaybg iio-sensor-proxy python-pyqt6 kscreen
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

Put your images here (names can differ, but then edit the script):

```
/home/myname/Pictures/landscape.jpg
/home/myname/Pictures/portrait.jpg
```

If you only have one image, point **both** variables at the same file.

---

## 4. Combined script

This one script:

- switches landscape / portrait on rotation
- paints wallpaper on every connected display
- restarts the LXQt panel and desktop after a monitor is plugged or unplugged
- re-applies the transparent desktop so the laptop does not stay black after unplugging externals

```bash
cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
LANDSCAPE="/home/myname/Pictures/landscape.jpg"
PORTRAIT="/home/myname/Pictures/portrait.jpg"
MODE="fill"
RECOVERING=0

outputs_key() {
    for s in /sys/class/drm/card*-*/status; do
        [ -f "$s" ] || continue
        echo "$(basename "$(dirname "$s")"):$(cat "$s")"
    done | sort | tr '\n' ' '
}

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

apply_wallpapers() {
    local img="$LANDSCAPE"
    [ "$1" = "portrait" ] && [ -f "$PORTRAIT" ] && img="$PORTRAIT"
    [ -f "$img" ] || return
    pkill -x swaybg 2>/dev/null
    sleep 0.3
    swaybg -i "$img" -m "$MODE" >/dev/null 2>&1 &
}

recover_desktop() {
    pkill -x lxqt-panel 2>/dev/null
    pkill -x pcmanfm-qt 2>/dev/null
    pkill -x swaybg 2>/dev/null
    sleep 1.2
    pcmanfm-qt --desktop >/dev/null 2>&1 &
    sleep 0.6
    pcmanfm-qt --set-wallpaper "$HOME/Pictures/transparent.svg" --wallpaper-mode stretch >/dev/null 2>&1
    lxqt-panel >/dev/null 2>&1 &
    sleep 2
    apply_wallpapers "$current_orient"
}

sleep 6

current_orient=$(get_orientation)
current_outs=$(outputs_key)
apply_wallpapers "$current_orient"

while true; do
    sleep 1.2
    new_orient=$(get_orientation)
    new_outs=$(outputs_key)

    if [ "$new_orient" != "$current_orient" ]; then
        current_orient="$new_orient"
        sleep 1.4
        apply_wallpapers "$current_orient"
    fi

    if [ -n "$new_outs" ] && [ "$new_outs" != "$current_outs" ] && [ "$RECOVERING" -eq 0 ]; then
        RECOVERING=1
        current_outs="$new_outs"
        sleep 3
        recover_desktop
        current_outs=$(outputs_key)
        RECOVERING=0
    fi
done
EOF
chmod +x ~/.local/bin/lxqt-orientation-wallpaper.sh
```

Change `myname` and the two picture paths to match your files.

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

Do **not** add a second hotplug watcher. This script already does that job.

---

## 6. Panel screen setting

While the panel is visible:

1. Right-click the panel → **Panel Settings**
2. Put the panel on the laptop screen (`eDP-1`), or on all screens if that option exists
3. Do not lock the panel only to `DP-1` / `DP-2`

If the panel lives only on an external monitor, unplugging that monitor will leave you with no panel until the script restarts it.

---

## 7. Test right now

```bash
rc-service iio-sensor-proxy status
pkill -f lxqt-orientation-wallpaper.sh 2>/dev/null
pkill -x swaybg 2>/dev/null
~/.local/bin/lxqt-orientation-wallpaper.sh &
```

Wait about 8 seconds, then:

- Fold the screen → wallpaper should switch
- Plug in a second display → panel may flicker once, wallpaper should appear on all screens
- Unplug the external → panel and desktop right-click should return on the laptop, wallpaper should return after a few seconds

---

## 8. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Nothing happens when folding | Sensor not running / KWin not rotating | Check `iio-sensor-proxy` and Display settings |
| Black background on all screens | Desktop not transparent, or swaybg not running | Re-apply the transparent SVG, then `pgrep -af swaybg` |
| Laptop stays black after unplugging externals | Transparent wallpaper was lost | Confirm `~/Pictures/transparent.svg` exists; script re-applies it |
| Panel gone after unplug | Panel was assigned only to the external | Set panel to laptop / all screens; wait 3–6 seconds for recover |
| Wrong image | Incorrect path in script | Edit `LANDSCAPE` / `PORTRAIT` |
| Wallpaper flashes a few times | Recover is restarting swaybg | Normal; the lock stops most double-fires |

Useful diagnostics:

```bash
rc-service iio-sensor-proxy status
pgrep -af lxqt-orientation-wallpaper
pgrep -af swaybg
ls -l ~/Pictures/transparent.svg ~/Pictures/landscape.jpg
```

Manual wallpaper test (should paint every current screen):

```bash
pkill -x swaybg 2>/dev/null
swaybg -i /home/myname/Pictures/landscape.jpg -m fill &
```

---

*Artix OpenRC only – no systemd.*
