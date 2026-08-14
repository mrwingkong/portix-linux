# 06 – Brightness OSD (adaptive + true 0 % = off)

Adaptive brightness with fine low-end control, true black at 0 %, separate SDR/HDR memory, and swaync OSD.

---

## Prerequisites (already done in 01 / 02 / 03)

These are normally required and were installed / configured in the earlier stages:

- Packages: `brightnessctl`, `kscreen`, `swaync`, `libnotify`, `sxhkd`, `iio-sensor-proxy`, `qt6-tools`
- User `myname` is in the `video` group
- PowerDevil brightness shortcuts already set to **None** (done in 02-post-install.md)
- `~/.local/bin` exists and is in `$PATH`
- LXQt notification daemon disabled and swaync running (guide 03)

If any are missing:

```bash
sudo pacman -S --needed brightnessctl kscreen swaync libnotify sxhkd iio-sensor-proxy qt6-tools
sudo usermod -aG video myname
```

**Log out completely and log back in** after adding the video group, then verify:

```bash
groups          # must contain "video"
brightnessctl -m
```

Also complete **03-swaync-notifications.md** first if you have not already.

---

## 1. Brightness OSD script

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
ID=2598
OUTPUT="eDP-1"                    # Change if kscreen-doctor -o shows a different name
HDR_STATE="/tmp/brightness-hdr"
SDR_STATE="/tmp/brightness-sdr"

# Tune this to ~90-95 % of your panel's peak nits
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
        kscreen-doctor output.${OUTPUT}.brightness.0 >/dev/null 2>&1
        kscreen-doctor output.${OUTPUT}.sdr-brightness.50 >/dev/null 2>&1
    else
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

## 2. Brightness mode watcher (restores last value when switching HDR ↔ SDR)

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

## 3. sxhkd keybindings

If you already created `~/.config/sxhkd/sxhkdrc` in the volume guide, append the brightness block. Otherwise create the file:

```bash
mkdir -p ~/.config/sxhkd
```

```bash
cat >> ~/.config/sxhkd/sxhkdrc << 'EOF'

# Brightness
XF86MonBrightnessUp
    ~/.local/bin/brightness-osd.sh up

XF86MonBrightnessDown
    ~/.local/bin/brightness-osd.sh down
EOF
```

---

## 4. Autostart for the mode watcher

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/brightness-mode-watcher.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Brightness Mode Watcher
Exec=$HOME/.local/bin/brightness-mode-watcher.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```

(sxhkd and swaync autostart were already added in earlier guides)

---

## 5. Start for current session

```bash
pkill -f brightness-mode-watcher 2>/dev/null
~/.local/bin/brightness-mode-watcher.sh &
pkill sxhkd 2>/dev/null
sxhkd &
```

---

## 6. Test

```bash
~/.local/bin/brightness-osd.sh up
~/.local/bin/brightness-osd.sh down
~/.local/bin/brightness-osd.sh set 30
```

Hardware brightness keys should now show a clean OSD and respect the adaptive steps + true 0 % = off.

---

## Notes

- Change `OUTPUT="eDP-1"` if `kscreen-doctor -o` shows a different name.
- Tune `HIGH_SDR=560` to roughly 90-95 % of your panel’s peak nits for best HDR behaviour.
- PowerDevil still handles idle dimming / screen-off / suspend – only the brightness keys are owned by this script.

---

*Artix OpenRC only – no systemd.*
