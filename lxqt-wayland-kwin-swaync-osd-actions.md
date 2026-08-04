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
brightnessctl pipewire pipewire-pulse wireplumber alsa-utils

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

### Volume (smooth 5 % steps, hard limit 100 %)

Uses `wpctl` (WirePlumber). This is the control that actually changes the audible volume.

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

### Brightness (SDR + HDR – adaptive steps, true peak, accurate 0–100 %)

Features:

- **SDR**: real backlight via `brightnessctl`, fine 1–2 % steps at the low end, true 0 % = off
- **HDR**: overall brightness % + unlocked high `sdr-brightness` (real panel peak)
- Separate memory for SDR / HDR so mode switches restore the last value
- Adaptive step sizes (finer near 0 %, larger higher up) so 0→100 does not take forever
- `set <percent>` supported for absolute values

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
        set)
            NEW="$2"
            [[ "$NEW" =~ ^[0-9]+$ ]] || exit 1
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

## 4. ALSA Speaker / Headphone Defaults

On many Intel sof-hda laptops the LXQt volume mixer shows three ALSA channels:

- `Master`
- `Speaker`
- `Headphone`

Desired behaviour:

- **Speaker** and **Headphone** always sit at **100 %** (hardware path full open)
- Real audible volume is controlled only by the keyboard OSD (`wpctl`)
- On headphone jack plug, Headphone is forced back to 100 % if the driver resets it

### Boot reset

```bash
cat > ~/.local/bin/volume-reset.sh << 'EOF'
#!/bin/bash
sleep 5
amixer -c 0 -q sset Speaker 100%
amixer -c 0 -q sset Headphone 100%
amixer -c 0 -q sset Master 0%
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0% 2>/dev/null
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0 2>/dev/null
EOF
chmod +x ~/.local/bin/volume-reset.sh
```

### Jack watcher (re-apply Headphone / Speaker 100 % on plug events)

```bash
cat > ~/.local/bin/headphone-fix.sh << 'EOF'
#!/bin/bash
# Only touches ALSA hardware channels. Never changes the real wpctl volume.
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

## 5. Brightness Mode Watcher (HDR ↔ SDR memory)

Restores the last used brightness when the display switches between HDR and SDR.

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

## 6. sxhkd Keybindings

```bash
mkdir -p ~/.config/sxhkd

# Append or merge these lines into ~/.config/sxhkd/sxhkdrc
cat >> ~/.config/sxhkd/sxhkdrc << 'EOF'

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

Reload:

```bash
killall sxhkd ; sxhkd &
```

> **Note:** While an LXQt volume dropdown is open, media keys may be blocked by KWin focus grab. This is normal. Close the dropdown or use the panel slider.

---

## 7. Autostart Entries

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

# Brightness mode watcher
cat > ~/.config/autostart/brightness-mode-watcher.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Brightness Mode Watcher
Exec=/home/USER/.local/bin/brightness-mode-watcher.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF

# Volume reset (Speaker/Headphone 100 %, real volume 0 %)
cat > ~/.config/autostart/volume-reset.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Volume Reset
Exec=/home/USER/.local/bin/volume-reset.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF

# Headphone jack fixer
cat > ~/.config/autostart/headphone-fix.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Headphone Volume Fix
Exec=/home/USER/.local/bin/headphone-fix.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```

Replace `USER` with your username (or use `$HOME` expansion if you prefer).

---

## 8. Panel Peak Tuning (HDR)

Find your panel’s real peak:

```bash
# Enable HDR first, then:
kscreen-doctor -o | grep -A25 eDP-1
```

Look for:

```
Peak brightness: 604 nits
Max average brightness: 497 nits
```

Set `HIGH_SDR` in `brightness-osd.sh` to roughly **90–95 % of Peak** (e.g. 560 for a 604-nit panel). Too high can clip; too low wastes headroom.

Optional one-shot unlock of the floor:

```bash
kscreen-doctor output.eDP-1.minBrightnessOverride.0
```

---

## 9. Orientation Wallpaper (optional)

If you use different wallpapers for landscape / portrait (convertible tablet mode):

```bash
# See the companion script lxqt-orientation-wallpaper.sh
# It switches wallpaper based on screen width vs height.
```

---

## 10. Notes & Known Limitations

| Topic | Behaviour |
|-------|-----------|
| Volume OSD | Smooth 5 % steps via `wpctl`. Hard ceiling 100 %. |
| ALSA Speaker / Headphone | Forced to 100 % on boot and on jack events so hardware path stays open. |
| Brightness SDR | True 0 % = backlight off. Fine steps at low end. |
| Brightness HDR | 0 % = darkest KWin allows; 100 % uses unlocked panel peak (`HIGH_SDR`). |
| Mode memory | Switching HDR ↔ SDR restores the last brightness used in that mode. |
| LXQt volume dropdown open | Media keys may be blocked by KWin until the popup is closed. |
| Global shortcut service | Incomplete on pure LXQt + KWin + OpenRC; sxhkd is the reliable method. |
| Tablet floating controls | Experimental; not included in this stable guide. |

---

## 11. Quick Verification

```bash
# Volume
~/.local/bin/volume-osd.sh up
~/.local/bin/volume-osd.sh down

# Brightness
~/.local/bin/brightness-osd.sh up
~/.local/bin/brightness-osd.sh down
~/.local/bin/brightness-osd.sh set 40

# ALSA levels
amixer -c 0 sget Speaker
amixer -c 0 sget Headphone
amixer -c 0 sget Master

# Current PipeWire volume
wpctl get-volume @DEFAULT_AUDIO_SINK@
```

---

*Last updated: August 2026 – Artix / OpenRC + LXQt + KWin Wayland*
