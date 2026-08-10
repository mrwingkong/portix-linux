# Volume OSD Setup (wpctl + sxhkd + swaync)

Smooth 5 % volume steps with proper OSD notifications and correct ALSA hardware levels for Intel SOF laptops.

## 1. Packages

```bash
sudo pacman -S --needed sxhkd pipewire pipewire-pulse wireplumber alsa-utils libnotify
```

## 2. Volume OSD Script

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

## 3. Volume Reset Script (critical for Intel SOF)

Forces Speaker + Headphone hardware channels to 100 % so the real volume control works.

```bash
cat > ~/.local/bin/volume-reset.sh << 'EOF'
#!/bin/bash
sleep 5
amixer -c 0 -q sset Speaker 100% unmute
amixer -c 0 -q sset Headphone 100% unmute
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0% 2>/dev/null
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0 2>/dev/null
EOF
chmod +x ~/.local/bin/volume-reset.sh
```

## 4. sxhkd Keybinds

```bash
mkdir -p ~/.config/sxhkd
cat >> ~/.config/sxhkd/sxhkdrc << 'EOF'

# Volume
XF86AudioRaiseVolume
    ~/.local/bin/volume-osd.sh up

XF86AudioLowerVolume
    ~/.local/bin/volume-osd.sh down

XF86AudioMute
    ~/.local/bin/volume-osd.sh mute
EOF
```

## 5. Autostart

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/sxhkd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=sxhkd
Exec=sxhkd
X-GNOME-Autostart-enabled=true
EOF

cat > ~/.config/autostart/volume-reset.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Volume Reset
Exec=$HOME/.local/bin/volume-reset.sh
X-GNOME-Autostart-enabled=true
NoDisplay=true
EOF
```

## 6. LXQt Volume Applet (for USB-C headphones)

1. Right-click volume icon → **Configure “Volume Control”**
2. Set backend to **PulseAudio**
3. Select the correct device when USB-C headphones are plugged in

## 7. Start for current session

```bash
pkill sxhkd 2>/dev/null
sxhkd &
~/.local/bin/volume-reset.sh
```

## 8. Test

```bash
~/.local/bin/volume-osd.sh up
~/.local/bin/volume-osd.sh down
wpctl get-volume @DEFAULT_AUDIO_SINK@
amixer -c 0 sget Speaker
```

## Notes

- On many modern Intel laptops the ALSA Speaker/Headphone controls must stay at 100 % or you will hear no sound even when the PipeWire volume is high.
- The tray icon may not update when using the PulseAudio backend. The OSD is the reliable feedback.
