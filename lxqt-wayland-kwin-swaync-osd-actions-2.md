# LXQt + KWin Wayland – OSD Actions, Brightness & Volume

Clean, working setup for **Artix / OpenRC** (also works on Arch) using:

- LXQt desktop
- KWin Wayland compositor
- swaync for OSD notifications
- sxhkd for media keys
- PipeWire + WirePlumber (`wpctl`)
- Adaptive brightness (SDR + HDR) with fine low-end control

This is the final remastered guide matching the scripts currently in use.

---

## 1. Required Packages

```bash
sudo pacman -S lxqt lxqt-wayland-session kwin kscreen kscreenlocker plasma-desktop systemsettings powerdevil layer-shell-qt swaync libnotify brightnessctl pipewire pipewire-pulse wireplumber sxhkd iio-sensor-proxy alsa-utils
```

Optional but useful:

```bash
sudo pacman -S fprintd python-pyqt6 qt6-tools xdg-desktop-portal xdg-desktop-portal-kde
```

---

## 2. Set KWin as Compositor

```bash
mkdir -p ~/.config/lxqt
cat > ~/.config/lxqt/session.conf << 'EOF'
[General]
compositor=kwin_wayland
EOF
```

---

## 3. OSD Scripts

```bash
mkdir -p ~/.local/bin
```

### 3.1 Volume OSD (`volume-osd.sh`)

```bash
cat > ~/.local/bin/volume-osd.sh << 'EOF'
#!/bin/bash
ID=2597
case "$1" in
    up)
        wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
        wpctl set-volume -l 1.0 @DEFAULT_AUDIO_SINK@ 5%+
        ;;
    down)
        wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
        wpctl set-volume @DEFAULT_AUDIO_SINK@ 5%-
        ;;
    mute)
        wpctl set-mute @DEFAULT_AUDIO_SINK@ toggle
        ;;
esac
VOLUME=$(wpctl get-volume @DEFAULT_AUDIO_SINK@ | awk '{print int($2 * 100)}')
MUTED=$(wpctl get-volume @DEFAULT_AUDIO_SINK@ | grep -q MUTED && echo 1 || echo 0)
if [ "$MUTED" = "1" ]; then
    notify-send -r $ID -u low -t 1500 \
        -h string:x-canonical-private-synchronous:volume \
        "Volume" "Muted"
else
    notify-send -r $ID -u low -t 1500 \
        -h int:value:"$VOLUME" \
        -h string:x-canonical-private-synchronous:volume \
        "Volume" "${VOLUME}%"
fi
EOF
chmod +x ~/.local/bin/volume-osd.sh
```

### 3.2 Brightness OSD (`brightness-osd.sh`)

Adaptive steps, true 0 % (off in SDR, darkest possible in HDR), unlocks panel peak in HDR.

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
ID=2598
OUTPUT="eDP-1"
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"
# Your panel peaks at 604 nits. 550–580 is a sweet spot for max without clipping.
HIGH_SDR=560

get_sdr_percent() {
    local p
    p=$(brightnessctl -m 2>/dev/null | awk -F, '{gsub(/%/,"",$4); print $4}' | head -1)
    [[ "$p" =~ ^[0-9]+$ ]] && echo "$p" || echo "20"
}

get_hdr_percent() {
    local p
    p=$(kscreen-doctor -o 2>/dev/null | grep -A30 "$OUTPUT" | grep -oP 'Brightness control: supported, set to \K[0-9]+' | head -1)
    [[ "$p" =~ ^[0-9]+$ ]] && echo "$p" || echo "50"
}

if kscreen-doctor -o 2>/dev/null | grep -A20 "$OUTPUT" | grep -q "SDR brightness:"; then
    # ---------- HDR ----------
    STATE_FILE="$HDR_STATE"
    CURRENT=$(get_hdr_percent)
    [ -f "$STATE_FILE" ] && CURRENT=$(cat "$STATE_FILE" 2>/dev/null || echo "$CURRENT")
    [[ "$CURRENT" =~ ^[0-9]+$ ]] || CURRENT=50

    case "$1" in
        up)
            if [ "$CURRENT" -lt 5 ]; then NEW=$((CURRENT + 1))
            elif [ "$CURRENT" -lt 15 ]; then NEW=$((CURRENT + 2))
            elif [ "$CURRENT" -lt 40 ]; then NEW=$((CURRENT + 3))
            else NEW=$((CURRENT + 5))
            fi
            ;;
        down)
            if [ "$CURRENT" -le 1 ]; then NEW=0
            elif [ "$CURRENT" -lt 5 ]; then NEW=$((CURRENT - 1))
            elif [ "$CURRENT" -lt 15 ]; then NEW=$((CURRENT - 2))
            elif [ "$CURRENT" -lt 40 ]; then NEW=$((CURRENT - 3))
            else NEW=$((CURRENT - 5))
            fi
            ;;
        set)
            NEW="$2"
            [[ "$NEW" =~ ^[0-9]+$ ]] || exit 1
            ;;
        *) exit 0 ;;
    esac

    [ "$NEW" -lt 0 ] && NEW=0
    [ "$NEW" -gt 100 ] && NEW=100

    if [ "$NEW" -eq 0 ]; then
        kscreen-doctor output.${OUTPUT}.brightness.0 >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.sdr-brightness.50 >/dev/null 2>&1
    else
        kscreen-doctor output.${OUTPUT}.sdr-brightness.${HIGH_SDR} >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.brightness.${NEW} >/dev/null 2>&1
    fi
    echo "$NEW" > "$STATE_FILE"
    PERCENT=$NEW
else
    # ---------- SDR ----------
    STATE_FILE="$SDR_STATE"
    CURRENT=$(get_sdr_percent)

    case "$1" in
        up)
            if [ "$CURRENT" -lt 8 ]; then brightnessctl set +1%
            elif [ "$CURRENT" -lt 20 ]; then brightnessctl set +2%
            elif [ "$CURRENT" -lt 40 ]; then brightnessctl set +3%
            else brightnessctl set +5%
            fi
            ;;
        down)
            if [ "$CURRENT" -le 1 ]; then brightnessctl set 0%
            elif [ "$CURRENT" -lt 8 ]; then brightnessctl set 1%-
            elif [ "$CURRENT" -lt 20 ]; then brightnessctl set 2%-
            elif [ "$CURRENT" -lt 40 ]; then brightnessctl set 3%-
            else brightnessctl set 5%-
            fi
            ;;
        set)
            NEW="$2"
            [[ "$NEW" =~ ^[0-9]+$ ]] || exit 1
            [ "$NEW" -lt 0 ] && NEW=0
            [ "$NEW" -gt 100 ] && NEW=100
            brightnessctl set "${NEW}%"
            ;;
        *) exit 0 ;;
    esac

    PERCENT=$(get_sdr_percent)
    echo "$PERCENT" > "$STATE_FILE"
fi

[ "$PERCENT" -lt 0 ] && PERCENT=0
[ "$PERCENT" -gt 100 ] && PERCENT=100

notify-send -r $ID -u low -t 1500 \
    -h int:value:"$PERCENT" \
    -h string:x-canonical-private-synchronous:brightness \
    "Brightness" "${PERCENT}%"
EOF
chmod +x ~/.local/bin/brightness-osd.sh
```

> **Tune these two values for your machine**
> - `OUTPUT="eDP-1"` → check with `kscreen-doctor -o`
> - `HIGH_SDR=560` → set to ~90–95 % of your panel’s peak nits (see section 11)

---

## 4. Brightness Mode Watcher

Restores the last brightness when switching between HDR and SDR.

```bash
cat > ~/.local/bin/brightness-mode-watcher.sh << 'EOF'
#!/bin/bash
OUTPUT="eDP-1"
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"
LAST_MODE=""

while true; do
    if kscreen-doctor -o 2>/dev/null | grep -A20 "$OUTPUT" | grep -q "SDR brightness:"; then
        MODE="hdr"
    else
        MODE="sdr"
    fi

    if [ "$MODE" != "$LAST_MODE" ]; then
        if [ "$MODE" = "hdr" ]; then
            TARGET=$(cat "$HDR_STATE" 2>/dev/null || echo 50)
            kscreen-doctor output.${OUTPUT}.sdr-brightness.560 >/dev/null 2>&1
            kscreen-doctor output.${OUTPUT}.brightness.${TARGET} >/dev/null 2>&1
        else
            TARGET=$(cat "$SDR_STATE" 2>/dev/null || echo 20)
            brightnessctl set ${TARGET}% >/dev/null 2>&1
        fi
        LAST_MODE="$MODE"
    fi

    sleep 2
done
EOF
chmod +x ~/.local/bin/brightness-mode-watcher.sh
```

---

## 5. Volume Reset on Login

Hardware ALSA channels (Speaker / Headphone) start at 100 %, real audible volume starts at 0 %.

```bash
cat > ~/.local/bin/volume-reset.sh << 'EOF'
#!/bin/bash
sleep 5
# Hardware channels full
amixer -c 0 -q sset Speaker 100%
amixer -c 0 -q sset Headphone 100%
# Real audible volume starts at 0
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0% 2>/dev/null
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0 2>/dev/null
EOF
chmod +x ~/.local/bin/volume-reset.sh
```

---

## 6. Optional but Recommended Helpers

### 6.1 Orientation / Wallpaper switcher

Switches wallpaper when the screen rotates (landscape ↔ portrait).

```bash
cat > ~/.local/bin/lxqt-orientation-wallpaper.sh << 'EOF'
#!/bin/bash
LANDSCAPE="/home/mrwingkong/Pictures/7680x2160.png"
PORTRAIT="/home/mrwingkong/Pictures/1440x2960.png"
MODE="fill"
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
    if [ -n "$SWAYBG_PID" ] && kill -0 "$SWAYBG_PID" 2>/dev/null; then
        kill "$SWAYBG_PID" 2>/dev/null
        wait "$SWAYBG_PID" 2>/dev/null || true
    fi
    pkill -x swaybg 2>/dev/null || true
    sleep 0.15
    swaybg -i "$img" -m "$MODE" &
    SWAYBG_PID=$!
}

sleep 2
current=$(get_orientation)
set_wallpaper "$current"

while true; do
    sleep 0.8
    new=$(get_orientation)
    if [ "$new" != "$current" ]; then
        current="$new"
        set_wallpaper "$current"
    fi
done
EOF
chmod +x ~/.local/bin/lxqt-orientation-wallpaper.sh
```

> Change the two image paths to your own wallpapers.

### 6.2 Battery low warning + auto-suspend

```bash
cat > ~/.local/bin/battery-watch.sh << 'EOF'
#!/bin/bash
LOW_LEVEL=5
WARN_LEVEL=10
WARNING_SECONDS=30
CHECK_INTERVAL=30
IGNORE_FILE="/tmp/battery-ignore"
WARNED_10_FILE="/tmp/battery-warned-10"

while true; do
    if [ -f "$IGNORE_FILE" ]; then
        sleep $CHECK_INTERVAL
        continue
    fi

    BAT_PATH=$(ls /sys/class/power_supply/ | grep -E 'BAT|BAT0|BAT1' | head -1)
    [ -z "$BAT_PATH" ] && sleep $CHECK_INTERVAL && continue

    CAPACITY=$(cat /sys/class/power_supply/$BAT_PATH/capacity 2>/dev/null)
    STATUS=$(cat /sys/class/power_supply/$BAT_PATH/status 2>/dev/null)

    if [ "$STATUS" != "Discharging" ] || [ "$CAPACITY" -gt "$WARN_LEVEL" ]; then
        rm -f "$WARNED_10_FILE"
    fi

    if [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -le "$WARN_LEVEL" ] && [ "$CAPACITY" -gt "$LOW_LEVEL" ]; then
        if [ ! -f "$WARNED_10_FILE" ]; then
            notify-send -u normal -t 5000 \
                -h string:x-canonical-private-synchronous:battery \
                "Battery Warning" "Battery is at ${CAPACITY}%"
            touch "$WARNED_10_FILE"
        fi
    fi

    if [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -le "$LOW_LEVEL" ]; then
        ACTION=$(notify-send -u normal -t 0 \
            -h string:x-canonical-private-synchronous:battery \
            -A "ignore=Ignore until reboot" \
            "Battery Low" "Battery is at ${CAPACITY}%.\nSuspending in ${WARNING_SECONDS} seconds...")

        if [ "$ACTION" = "ignore" ]; then
            touch "$IGNORE_FILE"
            notify-send -u normal -t 4000 "Suspend Cancelled" "Ignored until next login."
            sleep 300
            continue
        fi

        for ((i=WARNING_SECONDS; i>0; i--)); do
            sleep 1
            NEW_STATUS=$(cat /sys/class/power_supply/$BAT_PATH/status 2>/dev/null)
            if [ "$NEW_STATUS" != "Discharging" ]; then
                notify-send -u normal -t 3000 "Suspend Cancelled" "Charger detected."
                break
            fi
            if [ -f "$IGNORE_FILE" ]; then
                notify-send -u normal -t 3000 "Suspend Cancelled" "Ignored until reboot."
                break
            fi
            if [ $i -eq 1 ]; then
                loginctl suspend
            fi
        done
        sleep 300
    fi
    sleep $CHECK_INTERVAL
done
EOF
chmod +x ~/.local/bin/battery-watch.sh
```

### 6.3 Auto-close noisy notifications

```bash
cat > ~/.local/bin/nm-notif-closer.sh << 'EOF'
#!/bin/bash
dbus-monitor --session "interface='org.freedesktop.Notifications',member=Notify" |
while read -r line; do
    if echo "$line" | grep -qE 'string "(NetworkManager Applet|Removable media/devices manager)"'; then
        (
            sleep 3
            swaync-client --close-latest 2>/dev/null
        ) &
    fi
done
EOF
chmod +x ~/.local/bin/nm-notif-closer.sh
```

### 6.4 Headphone / Speaker ALSA keeper (optional)

Only needed if the ALSA Headphone/Speaker channels keep dropping from 100 %.

```bash
cat > ~/.local/bin/headphone-fix.sh << 'EOF'
#!/bin/bash
# Only re-applies ALSA Headphone/Speaker levels. Never touches the real volume.
pactl subscribe 2>/dev/null | while read -r line; do
    case "$line" in
        *change*|*new*)
            sleep 0.5
            amixer -c 0 -q sset Headphone 100%
            amixer -c 0 -q sset Speaker 100%
            ;;
    esac
done
EOF
chmod +x ~/.local/bin/headphone-fix.sh
```

---

## 7. sxhkd Keybinds

```bash
mkdir -p ~/.config/sxhkd
cat > ~/.config/sxhkd/sxhkdrc << 'EOF'
# Volume
XF86AudioRaiseVolume
    ~/.local/bin/volume-osd.sh up

XF86AudioLowerVolume
    ~/.local/bin/volume-osd.sh down

XF86AudioMute
    ~/.local/bin/volume-osd.sh mute

# Brightness
XF86MonBrightnessUp
    ~/.local/bin/brightness-osd.sh up

XF86MonBrightnessDown
    ~/.local/bin/brightness-osd.sh down
EOF
```

---

## 8. Autostart Entries

```bash
mkdir -p ~/.config/autostart

# swaync
cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=swaync
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF

# sxhkd
cat > ~/.config/autostart/sxhkd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=sxhkd
Exec=sxhkd
X-GNOME-Autostart-enabled=true
EOF

# PowerDevil (idle / lock / battery / lid)
cat > ~/.config/autostart/powerdevil.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PowerDevil
Exec=/usr/lib/org_kde_powerdevil
X-GNOME-Autostart-enabled=true
EOF

# Brightness mode watcher
cat > ~/.config/autostart/brightness-watcher.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Brightness Mode Watcher
Exec=/home/mrwingkong/.local/bin/brightness-mode-watcher.sh
X-GNOME-Autostart-enabled=true
EOF

# Volume reset (hardware 100 % / real volume 0 %)
cat > ~/.config/autostart/volume-reset.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Volume Reset
Exec=/home/mrwingkong/.local/bin/volume-reset.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF

# Orientation wallpaper (optional)
cat > ~/.config/autostart/orientation-wallpaper.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Orientation Wallpaper
Exec=/home/mrwingkong/.local/bin/lxqt-orientation-wallpaper.sh
X-GNOME-Autostart-enabled=true
EOF

# Battery watch (optional)
cat > ~/.config/autostart/battery-watch.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Battery Watch
Exec=/home/mrwingkong/.local/bin/battery-watch.sh
X-GNOME-Autostart-enabled=true
EOF

# Notification closer (optional)
cat > ~/.config/autostart/nm-notif-closer.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=NM Notif Closer
Exec=/home/mrwingkong/.local/bin/nm-notif-closer.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```

> Replace `/home/mrwingkong` with your own username if different.

---

## 9. Disable PowerDevil Brightness Shortcuts

PowerDevil must **not** own the brightness keys (otherwise it fights the scripts).

1. Open **System Settings → Shortcuts**
2. Search for **brightness**
3. Set these to **None**:
   - Increase Screen Brightness
   - Decrease Screen Brightness
   - Increase Screen Brightness by 1 %
   - Decrease Screen Brightness by 1 %

PowerDevil still handles idle, lock, battery and lid close.

---

## 10. LXQt Volume Applet

Use the **PulseAudio** backend so USB-C headphones appear in the list.

1. Right-click the volume icon → **Configure “Volume Control”**
2. Set backend to **PulseAudio**
3. Select the correct device (Speaker or the USB-C device when plugged in)

---

## 11. Finding Your Panel Peak (for `HIGH_SDR`)

With HDR enabled:

```bash
kscreen-doctor -o | grep -A40 eDP-1
```

Look for:

```
Peak brightness: 604 nits
```

Set `HIGH_SDR` in `brightness-osd.sh` to roughly 90–95 % of that value (e.g. `560` for a 604-nit panel).

---

## 12. Restart Everything (current session)

```bash
killall sxhkd swaync brightness-mode-watcher.sh lxqt-orientation-wallpaper.sh \
        battery-watch.sh nm-notif-closer.sh headphone-fix.sh 2>/dev/null

swaync &
sxhkd &
~/.local/bin/brightness-mode-watcher.sh &
~/.local/bin/lxqt-orientation-wallpaper.sh &
~/.local/bin/battery-watch.sh &
~/.local/bin/nm-notif-closer.sh &
# optional:
# ~/.local/bin/headphone-fix.sh &
```

After logout/login everything starts automatically via the autostart entries.

---

## 13. Final Behaviour

| Feature | Behaviour |
|--------|-----------|
| Volume keys | `wpctl` + swaync OSD, hard limit 100 % |
| Brightness keys (SDR) | Real backlight via `brightnessctl`, fine low-end steps, true 0 % = off |
| Brightness keys (HDR) | Overall brightness % + unlocked panel peak (`HIGH_SDR`) |
| 0 % in HDR | Darkest level KWin currently allows |
| 100 % in HDR | Near real hardware peak |
| HDR ↔ SDR switch | Automatically restores last used value |
| Login volume | ALSA Speaker/Headphone = 100 %, real volume = 0 % |
| USB-C headphones | Appear in LXQt volume applet (PulseAudio backend) |
| PowerDevil | Still handles idle, lock, battery, lid close |
| Notifications | Consistent look via swaync |

---

## 14. Tips & Known Limitations

- **True black in HDR** – KWin cannot fully extinguish the backlight; a very faint residual remains. This is a compositor limitation.
- **State files** – `/tmp/brightness-hdr` and `/tmp/brightness-sdr` are cleared on reboot; values are restored on the next key press or mode switch.
- **Multiple outputs** – Change `OUTPUT=` in the brightness scripts if you use an external monitor as primary.
- **ALSA vs PulseAudio** – Keep the LXQt volume applet on PulseAudio so USB devices appear. The tray icon may not always update; the OSD is the reliable feedback.
- **Artix / OpenRC** – There is no `systemctl --user`. All services are started via the `.desktop` files in `~/.config/autostart`.

---

*Remastered August 2026 – matches the final working scripts currently in use.*
