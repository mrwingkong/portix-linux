# LXQt + Labwc + Swaync Complete Setup Guide

A full working configuration for LXQt on Labwc (Wayland) featuring:

- Clean volume & brightness OSDs with progress bars
- Smart brightness (hardware backlight in SDR, gamma in HDR)
- Battery low warnings with “Ignore until reboot” button
- Auto-closing sticky WiFi and USB notifications
- HDR toggle that cleanly drops to TTY

---

## 1. Install Required Packages

```bash
sudo pacman -S swaync brightnessctl wireplumber alsa-utils bc glib2-devel base-devel wayland wlroots
```

---

## 2. Build and Install wlr-brightness

```bash
git clone https://github.com/mherzberg/wlr-brightness.git
cd wlr-brightness
git submodule update --init --recursive
make
sudo make install
```

---

## 3. Create Scripts Directory

```bash
mkdir -p ~/.local/bin
```

---

## 4. Volume Script

```bash
cat > ~/.local/bin/volume-osd.sh << 'EOF'
#!/bin/bash

ID=2597

case "$1" in
    up)
        amixer -D hw:0 set Master unmute
        amixer -D hw:0 set Master 5%+
        ;;
    down)
        amixer -D hw:0 set Master unmute
        amixer -D hw:0 set Master 5%-
        ;;
    mute)
        amixer -D hw:0 set Master toggle
        notify-send -r $ID -u low -t 1500 \
            -h string:x-canonical-private-synchronous:volume \
            "Volume" "Muted"
        exit 0
        ;;
esac

PERCENT=$(amixer -D hw:0 get Master | grep -o '[0-9]*%' | head -1 | tr -d '%')

notify-send -r $ID -u low -t 1500 \
    -h int:value:"$PERCENT" \
    -h string:x-canonical-private-synchronous:volume \
    "Volume" "${PERCENT}%"
EOF

chmod +x ~/.local/bin/volume-osd.sh
```

---

## 5. Smart Brightness Script

```bash
cat > ~/.local/bin/brightness-osd.sh << 'EOF'
#!/bin/bash
# ============================================================
# Smart Brightness OSD
# - HDR on  → uses wlr-brightness (gamma)
# - HDR off → uses brightnessctl (real backlight)
# ============================================================

ID=2598
RC="$HOME/.config/labwc/rc.xml"

if grep -q '<hdr>yes</hdr>' "$RC"; then
    # ----- HDR mode (gamma) -----
    STEP=0.05
    case "$1" in
        up)   gdbus call -e -d de.mherzberg -o /de/mherzberg/wlrbrightness -m de.mherzberg.wlrbrightness.increase $STEP > /dev/null ;;
        down) gdbus call -e -d de.mherzberg -o /de/mherzberg/wlrbrightness -m de.mherzberg.wlrbrightness.decrease $STEP > /dev/null ;;
    esac

    CURRENT=$(gdbus call -e -d de.mherzberg -o /de/mherzberg/wlrbrightness \
        -m de.mherzberg.wlrbrightness.get | grep -oP '[0-9.]+' | head -1)
    PERCENT=$(echo "$CURRENT * 100" | bc | cut -d. -f1)
else
    # ----- SDR mode (real backlight) -----
    case "$1" in
        up)   brightnessctl set +5% ;;
        down) brightnessctl set 5%- ;;
    esac
    PERCENT=$(brightnessctl -m | awk -F, '{print $4}' | tr -d '%')
fi

notify-send -r $ID -u low -t 1500 \
    -h int:value:"$PERCENT" \
    -h string:x-canonical-private-synchronous:brightness \
    "Brightness" "${PERCENT}%"
EOF

chmod +x ~/.local/bin/brightness-osd.sh
```

---

## 6. Battery Watcher Script

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

    # Reset the 10% warning flag when charging or above 10%
    if [ "$STATUS" != "Discharging" ] || [ "$CAPACITY" -gt "$WARN_LEVEL" ]; then
        rm -f "$WARNED_10_FILE"
    fi

    # 10% simple warning (only once)
    if [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -le "$WARN_LEVEL" ] && [ "$CAPACITY" -gt "$LOW_LEVEL" ]; then
        if [ ! -f "$WARNED_10_FILE" ]; then
            notify-send -u normal -t 5000 \
                -h string:x-canonical-private-synchronous:battery \
                "Battery Warning" "Battery is at ${CAPACITY}%"
            touch "$WARNED_10_FILE"
        fi
    fi

    # 5% critical warning + countdown
    if [ "$STATUS" = "Discharging" ] && [ "$CAPACITY" -le "$LOW_LEVEL" ]; then

        ACTION=$(notify-send -u critical -t 0 \
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

---

## 7. Sticky Notification Closer (WiFi + USB)

```bash
cat > ~/.local/bin/nm-notif-closer.sh << 'EOF'
#!/bin/bash
# Auto-close sticky notifications after 3 seconds
# - NetworkManager Applet (WiFi)
# - Removable media/devices manager (USB)

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

---

## 8. HDR Toggle Script

```bash
cat > ~/.local/bin/hdr-toggle.sh << 'EOF'
#!/bin/bash
RC="$HOME/.config/labwc/rc.xml"

if grep -q '<hdr>yes</hdr>' "$RC"; then
    sed -i 's/<hdr>yes<\/hdr>/<hdr>no<\/hdr>/' "$RC"
    # Reset gamma to full before leaving HDR
    gdbus call -e -d de.mherzberg -o /de/mherzberg/wlrbrightness \
        -m de.mherzberg.wlrbrightness.set 1.0 > /dev/null 2>&1
    notify-send -u normal -t 1500 "HDR" "Disabled – exiting to TTY..."
else
    sed -i 's/<hdr>no<\/hdr>/<hdr>yes<\/hdr>/' "$RC"
    notify-send -u normal -t 1500 "HDR" "Enabled – exiting to TTY..."
fi

sleep 0.8
labwc --exit
EOF

chmod +x ~/.local/bin/hdr-toggle.sh
```

---

## 9. Swaync Configuration

```bash
mkdir -p ~/.config/swaync
```

### config.json

```bash
cat > ~/.config/swaync/config.json << 'EOF'
{
  "$schema": "/usr/share/swaync/configSchema.json",
  "positionX": "right",
  "positionY": "bottom",
  "timeout": 2500,
  "timeout-low": 2000,
  "timeout-critical": 4000,
  "fit-to-screen": false,
  "notification-window-width": 300,
  "hide-on-clear": true,
  "widgets": ["title", "dnd", "notifications"],
  "rules": [
    {
      "app-name": "lxqt-panel",
      "summary": "Removable media/devices manager",
      "timeout": 3000
    },
    {
      "app-name": "lxqt-panel",
      "timeout": 3000
    },
    {
      "app-name": "*",
      "timeout": 2500
    }
  ]
}
EOF
```

### style.css

```bash
cat > ~/.config/swaync/style.css << 'EOF'
.notification {
  background: #000000;
  color: #ffffff;
  border-radius: 12px;
  border: 1px solid #555555;
  margin: 8px;
}

.notification-content {
  padding: 12px;
}

.notification .summary {
  font-weight: bold;
  font-size: 13px;
}

.notification .body {
  font-size: 12px;
}

.notification.low, .notification.normal {
  background: #000000;
  color: #ffffff;
  border: 1px solid #555555;
}

.notification.critical {
  background: #4a3c00;          /* dark yellow/brown */
  color: #ffdd57;               /* bright yellow text */
  border: 1px solid #ffdd57;
}
EOF
```

---

## 10. Labwc Configuration

### Environment

```bash
XCURSOR_THEME=FlatbedCursors-White
XCURSOR_SIZE=42
XKB_DEFAULT_LAYOUT=gb
WLR_RENDERER=vulkan
```

### Core section in `~/.config/labwc/rc.xml`

```xml
<core>
  <decoration>server</decoration>
  <gap>0</gap>
  <hdr>no</hdr>
</core>
```

### Keybinds (inside `<keyboard>`)

```xml
    <!-- Volume keys -->
    <keybind key="XF86_AudioLowerVolume">
      <action name="Execute" command="~/.local/bin/volume-osd.sh down" />
    </keybind>
    <keybind key="XF86_AudioRaiseVolume">
      <action name="Execute" command="~/.local/bin/volume-osd.sh up" />
    </keybind>
    <keybind key="XF86_AudioMute">
      <action name="Execute" command="~/.local/bin/volume-osd.sh mute" />
    </keybind>
    <!-- Brightness keys -->
    <keybind key="XF86MonBrightnessUp">
      <action name="Execute" command="~/.local/bin/brightness-osd.sh up" />
    </keybind>
    <keybind key="XF86MonBrightnessDown">
      <action name="Execute" command="~/.local/bin/brightness-osd.sh down" />
    </keybind>
        <!-- HDR Toggle -->
    <keybind key="W-h">
      <action name="Execute" command="~/.local/bin/hdr-toggle.sh" />
    </keybind>
  </keyboard>
```

### Autostart (`~/.config/labwc/autostart`)

```bash
# === labwc + LXQt autostart ===

# Background
#swaybg -i /usr/share/lxqt/wallpapers/origami-dark-labwc.png >/dev/null 2>&1 &

# GTK environment
dbus-update-activation-environment --systemd DISPLAY WAYLAND_DISPLAY > /dev/null 2>&1 &

# === Display configuration (persistent) ===
kanshi >/dev/null 2>&1 &

# Idle / screen power management (already good)
#swayidle -w timeout 300 "wlopm --off *" resume "wlopm --on *" > /dev/null 2>&1 &

wlr-brightness &

swaync &

# Audio defaults (delayed)
(
  sleep 4
  amixer -D hw:0 set Master unmute
  amixer -D hw:0 set Master 0%
  amixer -D hw:0 set Speaker unmute
  amixer -D hw:0 set Speaker 100%
  amixer -D hw:0 set Headphone unmute
  amixer -D hw:0 set Headphone 100%
) > /dev/null 2>&1 &

~/.local/bin/battery-watch.sh &

~/.local/bin/nm-notif-closer.sh &
```

---

## 11. One-time GUI Settings

1. Right-click the volume icon → **Volume Control Settings** → Uncheck **“Notify about volume changes with keyboard”**
2. **Power Management** → Battery tab → Set to **Suspend**
3. Removable Media plugin → Turn notifications **On**

---

## Important Notes

- Press `Super + H` to toggle HDR. You will be dropped to a clean TTY. Type `startlxqt` to return.
- In **SDR mode** you get full native brightness (hardware backlight).
- In **HDR mode** brightness uses gamma (desktop max is lower, but real HDR content can still peak higher).
- External HDR projectors use the same global HDR toggle.
- Labwc currently lacks an “SDR content brightness” control (unlike KWin).

