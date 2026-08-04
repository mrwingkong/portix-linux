# LXQt + KWin Wayland — Orientation-Aware Wallpaper (Tablet Mode)

**Persistent dual-wallpaper solution for landscape / portrait rotation on Artix (OpenRC) or any Arch-based system using LXQt Wayland + KWin.**

When you rotate into tablet/portrait mode, LXQt’s built-in wallpaper does not correctly re-layout. The image is cropped and black bars appear. This setup replaces the LXQt wallpaper with `swaybg` and automatically switches between two properly sized images on every rotation while **keeping desktop icons and the right-click menu**.

---

## Features

- Automatic switch between landscape and portrait wallpapers
- Survives reboots and session restarts
- Does **not** interfere with LXQt desktop icons or context menu
- Works with KWin’s tablet mode / auto-rotation
- Lightweight (one small bash + Python one-liner poll)
- No Plasma desktop required — pure LXQt + KWin

---

## Prerequisites

- LXQt Wayland session using **KWin** as compositor
- `swaybg`
- Python 3 + PyQt6 (already present with LXQt/KWin)
- Your two wallpapers:

```
Landscape : /home/mrwingkong/Pictures/7680x2160.png
Portrait  : /home/mrwingkong/Pictures/1440x2960.png
```

Install the missing package if needed:

```bash
sudo pacman -S swaybg
```

---

## 1. Make the LXQt desktop transparent

PCManFM-Qt (the LXQt desktop) must be transparent so `swaybg` can draw underneath it.

Create a 1×1 transparent SVG:

```bash
cat > ~/Pictures/transparent.svg << 'EOF'
<svg width="200" height="200" version="1.1"
     xmlns="http://www.w3.org/2000/svg"
     viewBox="0 0 200 200"/>
EOF
```

Apply it:

```bash
pcmanfm-qt --set-wallpaper ~/Pictures/transparent.svg --wallpaper-mode stretch
```

Or do it graphically:

1. Right-click the desktop → **Desktop Preferences**
2. Background tab → select the SVG
3. Wallpaper mode: Stretch (or any)

You can leave the solid black colour if you prefer, but the transparent SVG is cleaner.

---

## 2. Install the orientation script

```bash
mkdir -p ~/.local/bin
```

Create the script:

```bash
cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
# Persistent wallpaper switcher for LXQt + KWin Wayland
# Automatically switches between landscape and portrait images on rotation / tablet mode

LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
MODE="fill"          # fill | stretch | fit | center | tile
SWAYBG_PID=""

get_orientation() {
    # Uses Qt (already present on the system) — reliable on KWin Wayland
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

    # Kill previous instance cleanly
    if [ -n "$SWAYBG_PID" ] && kill -0 "$SWAYBG_PID" 2>/dev/null; then
        kill "$SWAYBG_PID" 2>/dev/null
        wait "$SWAYBG_PID" 2>/dev/null || true
    fi
    pkill -x swaybg 2>/dev/null || true
    sleep 0.15

    swaybg -i "$img" -m "$MODE" &
    SWAYBG_PID=$!
}

# Give the session a moment to settle
sleep 2
current=$(get_orientation)
set_wallpaper "$current"

# Continuous monitoring
while true; do
    sleep 0.8
    new=$(get_orientation)
    if [ "$new" != "$current" ]; then
        current="$new"
        set_wallpaper "$current"
    fi
done
EOF
```

Make it executable:

```bash
chmod +x ~/.local/bin/lxqt-orientation-wallpaper.sh
```

---

## 3. Autostart (persistent across reboots)

### Recommended method — LXQt Session Settings

1. Open **LXQt → Preferences → LXQt Settings → Session Settings**
2. Go to the **Autostart** tab
3. Click **Add**
4. Name: `Orientation Wallpaper`
5. Command: `/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh`
6. Leave “LXQt Autostart” checked if you only want it in LXQt sessions

### Alternative — Desktop file

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/lxqt-orientation-wallpaper.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Orientation Wallpaper
Comment=Auto-switch landscape/portrait wallpapers on rotation
Exec=/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh
OnlyShowIn=LXQt;
X-GNOME-Autostart-enabled=true
X-LXQt-Module=false
EOF
```

Log out and log back in (or reboot). The correct wallpaper should appear and switch cleanly on every rotation / tablet-mode change.

---

## 4. Optional tweaks

| Setting | Value | Notes |
|---------|-------|-------|
| `MODE` | `fill` | Crops to fill the screen (recommended) |
| | `stretch` | Distorts to exact size |
| | `fit` | Letterboxes if aspect differs |
| Poll interval | `0.8` | Change the `sleep 0.8` line if desired |
| Paths | Edit the two variables at the top of the script |

---

## Troubleshooting

**Wallpaper does not appear**
- Confirm the SVG is applied and the desktop is transparent
- Run the script manually in a terminal and watch for errors
- Check that `swaybg` is running: `pgrep -a swaybg`

**Wallpaper does not change on rotation**
- Make sure KWin is actually rotating the output (tablet mode enabled)
- Test the orientation helper manually:

```bash
python3 -c '
from PyQt6.QtGui import QGuiApplication
import sys
app = QGuiApplication(sys.argv)
s = app.primaryScreen()
print(s.size().width(), "x", s.size().height())
'
```

**Black bars still visible**
- The LXQt desktop is still drawing a solid colour. Re-apply the transparent SVG.

**Multiple swaybg processes**
- The script already kills previous instances. If any remain:

```bash
pkill -x swaybg
```

**Stop the daemon temporarily**

```bash
pkill -f lxqt-orientation-wallpaper
```

---

## How it works

1. PCManFM-Qt draws a fully transparent desktop surface (icons + menu still work).
2. `swaybg` paints the real wallpaper on the layer-shell background.
3. A tiny background script polls the logical screen size via Qt every ~0.8 s.
4. When width > height → landscape image; otherwise → portrait image.
5. On change it cleanly restarts `swaybg` with the matching file.

This approach is independent of the accelerometer and works whether rotation is triggered by tablet mode, manual KWin rotation, or any other method.

---

## License

Public domain / CC0 — do whatever you want with it.

---

*Tested on Artix OpenRC + LXQt Wayland + KWin (2026).*
