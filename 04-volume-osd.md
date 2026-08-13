# 04 – Volume OSD (swaync + sxhkd)

Volume keys with clean OSD notifications and correct ALSA hardware levels for Intel SOF laptops.

---

## Prerequisites (already done in 01 / 02)

These are normally required and were installed / configured in the earlier stages:

- Packages: `sxhkd`, `swaync`, `libnotify`, `pipewire`, `pipewire-pulse`, `wireplumber`, `alsa-utils`
- `~/.local/bin` exists and is in `$PATH`
- `volume-reset.sh` already created in 02-post-install.md
- User is in the `video` group (not strictly required for volume, but done for brightness)

If any are missing:

```bash
sudo pacman -S --needed sxhkd swaync libnotify pipewire pipewire-pulse wireplumber alsa-utils
mkdir -p ~/.local/bin
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## 1. Volume OSD script

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

## 2. sxhkd keybindings

```bash
mkdir -p ~/.config/sxhkd
```

```bash
cat > ~/.config/sxhkd/sxhkdrc << 'EOF'
# Volume
XF86AudioRaiseVolume
    ~/.local/bin/volume-osd.sh up

XF86AudioLowerVolume
    ~/.local/bin/volume-osd.sh down

XF86AudioMute
    ~/.local/bin/volume-osd.sh mute
EOF
```

(If you already have an sxhkdrc from the brightness guide, append the volume block instead of overwriting.)

---

## 3. Disable LXQt notification daemon (so swaync owns the bus)

```bash
killall lxqt-notifications 2>/dev/null
```

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/lxqt-notifications.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=LXQt Notification Daemon
Exec=lxqt-notifications
Hidden=true
X-GNOME-Autostart-enabled=false
EOF
```

---

## 4. Autostart entries

```bash
cat > ~/.config/autostart/sxhkd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=sxhkd
Exec=sxhkd
X-GNOME-Autostart-enabled=true
EOF
```

```bash
cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=swaync
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF
```

```bash
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

## 5. Start for current session

```bash
pkill sxhkd 2>/dev/null
pkill swaync 2>/dev/null
sxhkd &
swaync &
~/.local/bin/volume-reset.sh
```

---

## 6. LXQt Volume Applet (USB-C headphones)

1. Right-click the volume icon → **Configure “Volume Control”**
2. Set backend to **PulseAudio**
3. Select the correct device when USB-C headphones are plugged in

---

## 7. Test

```bash
~/.local/bin/volume-osd.sh up
~/.local/bin/volume-osd.sh down
~/.local/bin/volume-osd.sh mute
```

Hardware volume keys should now show a clean OSD via swaync.

---

*Artix OpenRC only – no systemd.*
