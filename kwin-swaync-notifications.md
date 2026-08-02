# LXQt + KWin Wayland Setup (Artix / OpenRC)

Complete guide for a minimal **LXQt** desktop using **KWin** as the Wayland compositor, with consistent OSD notifications via **swaync**.

This replaces the previous LabWC setup while keeping LXQt as the main environment.

---

## 1. Required Packages

```bash
# Core LXQt + KWin
lxqt lxqt-wayland-session
kwin kscreen kscreenlocker plasma-desktop systemsettings powerdevil layer-shell-qt

# Notifications / OSD
swaync libnotify

# Volume & Brightness tools
brightnessctl pamixer pipewire pipewire-pulse wireplumber

# Keybinding daemon
sxhkd

# Optional but recommended
fprintd
qt6-tools xdg-desktop-portal xdg-desktop-portal-kde iio-sensor-proxy
```

> Keep `labwc` installed if you want the choice at session start.

---

## 2. Set KWin as Default Compositor

```bash
mkdir -p ~/.config/lxqt
cat > ~/.config/lxqt/session.conf << 'EOF'
[General]
compositor=kwin_wayland
EOF
```

---

## 3. OSD Scripts

### Volume (hard limit at 100%)

```bash
mkdir -p ~/.local/bin

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

### Brightness (SDR + HDR with separate memory)

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
ID=2598
OUTPUT="eDP-1"          # Change if your output name is different
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"

if kscreen-doctor -o 2>/dev/null | grep -A20 "$OUTPUT" | grep -q "SDR brightness:"; then
    # ---------- HDR mode ----------
    STATE_FILE="$HDR_STATE"
    DEFAULT=240          # ~35%

    CURRENT=$(cat "$STATE_FILE" 2>/dev/null || echo $DEFAULT)

    case "$1" in
        up)   NEW=$((CURRENT + 25)) ;;
        down) NEW=$((CURRENT - 25)) ;;
    esac

    [ "$NEW" -lt 50 ]  && NEW=50
    [ "$NEW" -gt 600 ] && NEW=600

    kscreen-doctor output.${OUTPUT}.sdr-brightness.${NEW} >/dev/null 2>&1
    echo "$NEW" > "$STATE_FILE"

    PERCENT=$(( (NEW - 50) * 100 / 550 ))
else
    # ---------- SDR mode ----------
    STATE_FILE="$SDR_STATE"
    DEFAULT=20           # 20%

    CURRENT=$(cat "$STATE_FILE" 2>/dev/null || echo $DEFAULT)
    brightnessctl set ${CURRENT}% >/dev/null 2>&1

    case "$1" in
        up)   brightnessctl set +5% ;;
        down) brightnessctl set 5%- ;;
    esac

    PERCENT=$(brightnessctl -m | awk -F, '{print $4}' | tr -d '%')
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

> **Note:** Change `OUTPUT="eDP-1"` if `kscreen-doctor -o` shows a different name.

---

## 4. Brightness Mode Watcher

This automatically restores the last used brightness when you switch between HDR and SDR in System Settings.

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
            TARGET=$(cat "$HDR_STATE" 2>/dev/null || echo 240)
            kscreen-doctor output.${OUTPUT}.sdr-brightness.${TARGET} >/dev/null 2>&1
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

## 5. sxhkd Keybinds

```bash
mkdir -p ~/.config/sxhkd

cat > ~/.config/sxhkd/sxhkdrc << 'EOF'
XF86AudioRaiseVolume
    ~/.local/bin/volume-osd.sh up

XF86AudioLowerVolume
    ~/.local/bin/volume-osd.sh down

XF86AudioMute
    ~/.local/bin/volume-osd.sh mute

XF86MonBrightnessUp
    ~/.local/bin/brightness-osd.sh up

XF86MonBrightnessDown
    ~/.local/bin/brightness-osd.sh down
EOF
```

---

## 6. Autostart Entries

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=SwayNC
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF

cat > ~/.config/autostart/sxhkd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=sxhkd
Exec=sxhkd
X-GNOME-Autostart-enabled=true
EOF

cat > ~/.config/autostart/powerdevil.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PowerDevil
Exec=/usr/lib/org_kde_powerdevil
X-GNOME-Autostart-enabled=true
EOF

cat > ~/.config/autostart/brightness-watcher.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Brightness Mode Watcher
Exec=/home/%u/.local/bin/brightness-mode-watcher.sh
X-GNOME-Autostart-enabled=true
EOF
```

---

## 7. Important: Disable PowerDevil Brightness Shortcuts

PowerDevil must stay running for idle timeout, lock screen, battery management, etc.

But we must stop it from handling the brightness keys.

1. Open **System Settings → Shortcuts**
2. Search for **brightness**
3. Set these to **None**:
   - Increase Screen Brightness
   - Decrease Screen Brightness
   - Increase Screen Brightness by 1%
   - Decrease Screen Brightness by 1%

---

## 8. Final Result

| Feature                    | Behaviour                                      |
|---------------------------|------------------------------------------------|
| Volume keys               | Controlled by script + swaync OSD (max 100%)  |
| Brightness keys (SDR)     | Real backlight via `brightnessctl` + OSD      |
| Brightness keys (HDR)     | KWin SDR brightness via `kscreen-doctor` + OSD|
| Mode switching (HDR↔SDR)  | Automatically restores last used value        |
| PowerDevil                | Still handles idle, lock, battery, lid close  |
| Notifications             | Consistent look via swaync                    |

---

## 9. Restart Services (current session)

```bash
killall sxhkd swaync brightness-mode-watcher.sh 2>/dev/null
swaync &
sxhkd &
~/.local/bin/brightness-mode-watcher.sh &
```

After logout/login everything starts automatically.
```
