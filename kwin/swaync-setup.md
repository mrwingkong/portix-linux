# Disable LXQt notification daemon so swaync owns the bus

```bash
killall lxqt-notificationd 2>/dev/null
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/lxqt-notificationd.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=LXQt Notification Daemon
Exec=lxqt-notificationd
Hidden=true
X-GNOME-Autostart-enabled=false
EOF
```

# SwayNC Setup for LXQt + KWin Wayland

Lightweight notification daemon that provides consistent OSD notifications for volume, brightness, battery, etc.

## 1. Install

```bash
sudo pacman -S swaync libnotify
```

## 2. Autostart

```bash
mkdir -p ~/.config/autostart
cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=swaync
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF
```

## 3. Start for current session

```bash
pkill swaync 2>/dev/null
swaync &
```

## 4. Test

```bash
notify-send -u low -t 2000 "Test" "SwayNC is working"
```

## Notes

- Default config lives in `/etc/xdg/swaync/`
- You can copy and customise it later if desired:
  ```bash
  mkdir -p ~/.config/swaync
  cp /etc/xdg/swaync/config.json ~/.config/swaync/
  cp /etc/xdg/swaync/style.css ~/.config/swaync/
  ```
- swaync is required for the volume and brightness OSD scripts to show proper progress bars.
