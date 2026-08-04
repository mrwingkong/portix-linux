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

### Brightness (SDR + HDR – adaptive steps, full peak unlock)

This script provides:

- **SDR**: real backlight control via `brightnessctl` with fine steps at low brightness
- **HDR**: overall brightness % + unlocked high `sdr-brightness` (real panel peak)
- Separate memory for SDR / HDR so switching modes restores the last value
- Smooth adaptive step sizes (1 % at the bottom → larger steps higher up)
- 0 % maps to the darkest level KWin allows in HDR

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
ID=2598
OUTPUT="eDP-1"                    # Change if kscreen-doctor -o shows a different name
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"

# ============================================================
# Tune this value to your panel's peak.
# Check with:  kscreen-doctor -o   (while HDR is enabled)
# Look for "Peak brightness: XXX nits"
# Recommended starting point ≈ Peak × 0.90 – 0.95
# Example: Peak 604 nits → HIGH_SDR=560
# ============================================================
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
    # ---------- HDR mode ----------
    STATE_FILE="$HDR_STATE"
    CURRENT=$(get_hdr_percent)
    [ -f "$STATE_FILE" ] && CURRENT=$(cat "$STATE_FILE" 2>/dev/null || echo "$CURRENT")
    [[ "$CURRENT" =~ ^[0-9]+$ ]] || CURRENT=50

    case "$1" in
        up)
            if   [ "$CURRENT" -lt 5 ];  then NEW=$((CURRENT + 1))
            elif [ "$CURRENT" -lt 15 ]; then NEW=$((CURRENT + 2))
            elif [ "$CURRENT" -lt 40 ]; then NEW=$((CURRENT + 3))
            else                            NEW=$((CURRENT + 5))
            fi
            ;;
        down)
            if   [ "$CURRENT" -le 1 ];  then NEW=0
            elif [ "$CURRENT" -lt 5 ];  then NEW=$((CURRENT - 1))
            elif [ "$CURRENT" -lt 15 ]; then NEW=$((CURRENT - 2))
            elif [ "$CURRENT" -lt 40 ]; then NEW=$((CURRENT - 3))
            else                            NEW=$((CURRENT - 5))
            fi
            ;;
        *) exit 0 ;;
    esac

    [ "$NEW" -lt 0 ]   && NEW=0
    [ "$NEW" -gt 100 ] && NEW=100

    if [ "$NEW" -eq 0 ]; then
        # Darkest possible under KWin HDR
        kscreen-doctor output.${OUTPUT}.brightness.0 >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.sdr-brightness.50 >/dev/null 2>&1
    else
        # Unlock real peak + set overall brightness percentage
        kscreen-doctor output.${OUTPUT}.sdr-brightness.${HIGH_SDR} >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.brightness.${NEW} >/dev/null 2>&1
    fi

    echo "$NEW" > "$STATE_FILE"
    PERCENT=$NEW
else
    # ---------- SDR mode ----------
    STATE_FILE="$SDR_STATE"
    CURRENT=$(get_sdr_percent)

    case "$1" in
        up)
            if   [ "$CURRENT" -lt 8 ];  then brightnessctl set +1%
            elif [ "$CURRENT" -lt 20 ]; then brightnessctl set +2%
            elif [ "$CURRENT" -lt 40 ]; then brightnessctl set +3%
            else                            brightnessctl set +5%
            fi
            ;;
        down)
            if   [ "$CURRENT" -le 1 ];  then brightnessctl set 0%
            elif [ "$CURRENT" -lt 8 ];  then brightnessctl set 1%-
            elif [ "$CURRENT" -lt 20 ]; then brightnessctl set 2%-
            elif [ "$CURRENT" -lt 40 ]; then brightnessctl set 3%-
            else                            brightnessctl set 5%-
            fi
            ;;
        *) exit 0 ;;
    esac

    PERCENT=$(get_sdr_percent)
    echo "$PERCENT" > "$STATE_FILE"
fi

[ "$PERCENT" -lt 0 ]   && PERCENT=0
[ "$PERCENT" -gt 100 ] && PERCENT=100

notify-send -r $ID -u low -t 1500 \
    -h int:value:"$PERCENT" \
    -h string:x-canonical-private-synchronous:brightness \
    "Brightness" "${PERCENT}%"
EOF

chmod +x ~/.local/bin/brightness-osd.sh
```

> **Important notes**
>
> - Change `OUTPUT="eDP-1"` if `kscreen-doctor -o` shows a different connector name.
> - Adjust `HIGH_SDR=` to match your panel (see section 10 below).
> - In HDR, absolute 0 % is the darkest KWin currently allows (extremely faint residual light is common on laptop panels). True hardware-off is still limited by KWin’s HDR path.

---

## 4. Brightness Mode Watcher

Automatically restores the last used brightness when you switch between HDR and SDR in System Settings.

```bash
cat > ~/.local/bin/brightness-mode-watcher.sh << 'EOF'
#!/bin/bash
OUTPUT="eDP-1"
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"
HIGH_SDR=560                    # Keep in sync with brightness-osd.sh
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
            if [ "$TARGET" -eq 0 ]; then
                kscreen-doctor output.${OUTPUT}.brightness.0 >/dev/null 2>&1
                kscreen-doctor output.${OUTPUT}.sdr-brightness.50 >/dev/null 2>&1
            else
                kscreen-doctor output.${OUTPUT}.sdr-brightness.${HIGH_SDR} >/dev/null 2>&1
                kscreen-doctor output.${OUTPUT}.brightness.${TARGET} >/dev/null 2>&1
            fi
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

| Feature                    | Behaviour                                              |
|---------------------------|--------------------------------------------------------|
| Volume keys               | Controlled by script + swaync OSD (hard limit 100 %)  |
| Brightness keys (SDR)     | Real backlight via `brightnessctl`, fine low-end steps|
| Brightness keys (HDR)     | Overall brightness % + unlocked panel peak            |
| 0 % in HDR                | Darkest level KWin currently allows                   |
| 100 % in HDR              | Near real hardware peak (tunable via `HIGH_SDR`)      |
| Mode switching (HDR↔SDR)  | Automatically restores last used value                |
| PowerDevil                | Still handles idle, lock, battery, lid close          |
| Notifications             | Consistent look via swaync                            |

---

## 9. Restart Services (current session)

```bash
killall sxhkd swaync brightness-mode-watcher.sh 2>/dev/null
swaync &
sxhkd &
~/.local/bin/brightness-mode-watcher.sh &
```

After logout / login everything starts automatically.

---

## 10. Finding Your Panel’s Peak Brightness (recommended)

While HDR is **enabled**, run:

```bash
kscreen-doctor -o | grep -A40 eDP-1
```

Look for:

```
Peak brightness: 604 nits
Max average brightness: 497 nits
Min brightness: 0.0004 nits
```

Then set `HIGH_SDR` in both scripts to roughly 90–95 % of the Peak value (e.g. 560 for a 604-nit panel).

You can also inspect the raw EDID (manufacturer / model) with:

```bash
# Requires v4l-utils (or edid-decode)
edid-decode /sys/class/drm/card*-eDP-1/edid 2>/dev/null | head -60
```

---

## 11. Tips & Known Limitations

- **True black in HDR**: KWin on laptop panels currently cannot fully extinguish the backlight even at 0 %. The residual glow is a compositor limitation, not a hardware one (LabWC gamma could go fully black on the same panel).
- **Peak unlock**: Raising `sdr-brightness` is what actually unlocks the high nits; the overall brightness percentage then scales from that peak.
- **State files**: `/tmp/brightness-hdr` and `/tmp/brightness-sdr` are cleared on reboot (intentional). Values are restored on the next key press.
- **Multiple outputs**: Change `OUTPUT=` if you use an external monitor as primary.
