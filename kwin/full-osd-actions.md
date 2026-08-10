# LXQt + KWin Wayland – OSD Actions, Brightness & Volume (Final Working Guide)

Clean, battle-tested setup for **Artix / OpenRC** (also works on Arch) using:

- LXQt desktop
- KWin Wayland compositor
- swaync for OSD notifications
- sxhkd for media keys
- PipeWire + WirePlumber (`wpctl`)
- Adaptive brightness (SDR + HDR) with true 0 % = off and fine low-end control

This version includes every missing step discovered during real fresh-install testing (permissions, PowerDevil conflicts, ALSA levels, service startup, etc.).

---

## 0. Critical Prerequisites (do these first)

```bash
# Install everything needed
sudo pacman -S --needed \
  sxhkd brightnessctl swaync libnotify \
  pipewire pipewire-pulse wireplumber alsa-utils \
  kwin kscreen powerdevil systemsettings layer-shell-qt \
  iio-sensor-proxy qt6-tools

# Brightnessctl needs write access to the backlight
sudo usermod -aG video $USER
```

**Log out completely and log back in** (or reboot).  
Group membership only applies after a new login session.

Verify:

```bash
groups          # must contain "video"
brightnessctl -m
```

---

## 1. Set KWin as Default Compositor

```bash
mkdir -p ~/.config/lxqt
cat > ~/.config/lxqt/session.conf << 'EOF'
[General]
compositor=kwin_wayland
EOF
```

---

## 2. Create Script Directory + PATH

```bash
mkdir -p ~/.local/bin
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 3. Volume OSD Script

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
    set)
        VAL="$2"
        [[ "$VAL" =~ ^[0-9]+$ ]] || exit 1
        [ "$VAL" -gt 100 ] && VAL=100
        [ "$VAL" -lt 0 ] && VAL=0
        wpctl set-mute @DEFAULT_AUDIO_SINK@ 0
        wpctl set-volume @DEFAULT_AUDIO_SINK@ "${VAL}%"
        ;;
    *)
        exit 0
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

---

## 4. Brightness OSD Script (adaptive + true 0 % = off)

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
ID=2598
OUTPUT="eDP-1"                    # Change if kscreen-doctor -o shows a different name
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"

# Tune this to ~90-95 % of your panel's peak nits (see section 11)
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
        set)
            NEW="$2"
            [[ "$NEW" =~ ^[0-9]+$ ]] || exit 1
            ;;
        *) exit 0 ;;
    esac

    [ "$NEW" -lt 0 ]   && NEW=0
    [ "$NEW" -gt 100 ] && NEW=100

    if [ "$NEW" -eq 0 ]; then
        # True black in HDR
        kscreen-doctor output.${OUTPUT}.brightness.0 >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.sdr-brightness.50 >/dev/null 2>&1
    else
        # Unlock real panel peak + set overall brightness %
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

[ "$PERCENT" -lt 0 ]   && PERCENT=0
[ "$PERCENT" -gt 100 ] && PERCENT=100

notify-send -r $ID -u low -t 1500 \
    -h int:value:"$PERCENT" \
    -h string:x-canonical-private-synchronous:brightness \
    "Brightness" "${PERCENT}%"
EOF
chmod +x ~/.local/bin/brightness-osd.sh
```

---

## 5. Volume Reset (ALSA hardware levels)

On many Intel SOF laptops the real volume only works if Speaker/Headphone are at 100 %.

```bash
cat > ~/.local/bin/volume-reset.sh << 'EOF'
#!/bin/bash
sleep 5
# Hardware path fully open
amixer -c 0 -q sset Speaker 100% unmute
amixer -c 0 -q sset Headphone 100% unmute
# Real audible volume starts at 0
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0% 2>/dev/null
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0 2>/dev/null
EOF
chmod +x ~/.local/bin/volume-reset.sh
```

---

## 6. Brightness Mode Watcher (HDR ↔ SDR memory)

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

    if [ "$MODE" != "$LAST_MODE" ] && [ -n "$LAST_MODE" ]; then
        if [ "$MODE" = "hdr" ] && [ -f "$HDR_STATE" ]; then
            ~/.local/bin/brightness-osd.sh set "$(cat "$HDR_STATE")"
        elif [ "$MODE" = "sdr" ] && [ -f "$SDR_STATE" ]; then
            ~/.local/bin/brightness-osd.sh set "$(cat "$SDR_STATE")"
        fi
    fi
    LAST_MODE="$MODE"
    sleep 2
done
EOF
chmod +x ~/.local/bin/brightness-mode-watcher.sh
```

---

## 7. sxhkd Keybindings

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

## 8. Disable PowerDevil Brightness Shortcuts (CRITICAL)

PowerDevil will otherwise steal the brightness keys.

1. Open **System Settings → Shortcuts**
2. Search for **brightness**
3. Set these four to **None**:
   - Increase Screen Brightness
   - Decrease Screen Brightness
   - Increase Screen Brightness by 1 %
   - Decrease Screen Brightness by 1 %

PowerDevil still handles idle, lid close, lock and battery.

---

## 9. Autostart Entries

```bash
mkdir -p ~/.config/autostart

# sxhkd
cat > ~/.config/autostart/sxhkd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=sxhkd
Exec=sxhkd
X-GNOME-Autostart-enabled=true
EOF

# swaync
cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=swaync
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF

# PowerDevil
cat > ~/.config/autostart/powerdevil.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PowerDevil
Exec=/usr/lib/org_kde_powerdevil
X-GNOME-Autostart-enabled=true
EOF

# Brightness mode watcher
cat > ~/.config/autostart/brightness-mode-watcher.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Brightness Mode Watcher
Exec=$HOME/.local/bin/brightness-mode-watcher.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF

# Volume reset
cat > ~/.config/autostart/volume-reset.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Volume Reset
Exec=$HOME/.local/bin/volume-reset.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```

---

## 10. LXQt Volume Applet

For USB-C headphones to appear:

1. Right-click the volume icon → **Configure “Volume Control”**
2. Set backend to **PulseAudio**
3. Select the correct device when needed

Note: the tray icon may not update reliably on the PulseAudio backend (known LXQt limitation). The OSD remains the reliable feedback.

---

## 11. Panel Peak Tuning (HDR)

```bash
# Enable HDR first, then:
kscreen-doctor -o | grep -A25 eDP-1
```

Look for `Peak brightness: XXX nits` and set `HIGH_SDR` in the brightness script to roughly 90–95 % of that value.

---

## 12. Start Everything for the Current Session

After creating the files you must start the services once (or log out/in):

```bash
pkill sxhkd swaync 2>/dev/null

swaync &
sxhkd &
~/.local/bin/brightness-mode-watcher.sh &
~/.local/bin/volume-reset.sh &
```

---

## 13. Quick Verification

```bash
# Volume
~/.local/bin/volume-osd.sh up
~/.local/bin/volume-osd.sh down
wpctl get-volume @DEFAULT_AUDIO_SINK@

# Brightness
~/.local/bin/brightness-osd.sh up
~/.local/bin/brightness-osd.sh down
~/.local/bin/brightness-osd.sh set 40
brightnessctl -m
```

---

## 14. Known Limitations & Notes

| Topic | Behaviour |
|-------|-----------|
| Volume OSD | Smooth 5 % steps via `wpctl`. Hard ceiling 100 %. |
| ALSA Speaker / Headphone | Forced to 100 % on boot so the hardware path stays open. |
| Brightness SDR | True 0 % = backlight off. Fine steps at low end. |
| Brightness HDR | 0 % = fully black. 100 % uses unlocked panel peak. |
| Mode memory | Switching HDR ↔ SDR restores the last brightness used in that mode. |
| PowerDevil dimming | Can conflict with external brightness control. Disable “Dim screen” if restore fails. |
| Tray icon (PulseAudio) | Often does not update. OSD is the reliable indicator. |
| Global shortcuts | Incomplete on pure LXQt + KWin + OpenRC → sxhkd is required. |

---

*Final working version – August 2026 – Artix / OpenRC + LXQt + KWin Wayland*
```
