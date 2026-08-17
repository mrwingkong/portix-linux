# 03 – SwayNC Notifications

Disable the LXQt notification daemon and set up swaync so volume / brightness OSDs (and other notifications) work cleanly.

This guide must be completed **before** the volume and brightness OSD guides.

---

## Prerequisites (already done in 01 / 02)

- Packages `swaync` and `libnotify` were installed in 02-post-install.md
- `~/.config/autostart` directory may already exist

If missing:

```bash
sudo pacman -S --needed swaync libnotify
mkdir -p ~/.config/autostart
```

---

## 1. Disable LXQt notification daemon

LXQt ships its own notification daemon. We stop it and prevent it from starting so swaync can own the notification bus.

The official autostart file is named `lxqt-notifications.desktop` (the binary is still `lxqt-notificationd`).

```bash
killall lxqt-notificationd 2>/dev/null
```

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/lxqt-notifications.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=LXQt Notification Daemon
Exec=lxqt-notificationd
Hidden=true
X-GNOME-Autostart-enabled=false
EOF
```

---

## 2. Create local config folder (defaults, no custom changes)

Copy the system defaults into your home directory so you can edit them later without touching the system files.  
These are the stock files — behaviour stays exactly the same as the current GitHub guides.

```bash
mkdir -p ~/.config/swaync
```

```bash
cp /etc/xdg/swaync/config.json ~/.config/swaync/
cp /etc/xdg/swaync/style.css ~/.config/swaync/
```

You now have:

- `~/.config/swaync/config.json`
- `~/.config/swaync/style.css`

Edit them only when you want to customise appearance or behaviour later.

---

## 3. Autostart swaync

```bash
cat > ~/.config/autostart/swaync.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=swaync
Exec=swaync
X-GNOME-Autostart-enabled=true
EOF
```

---

## 4. Start for the current session

```bash
pkill swaync 2>/dev/null
swaync &
```

---

## 5. Test

```bash
notify-send -u low -t 2000 "Test" "SwayNC is working"
```

You should see a notification appear.

---

## Notes

- swaync is required for the volume and brightness OSD scripts to show proper progress bars.
- The local copies in `~/.config/swaync/` are identical to the system defaults on first install. Change them only when you want a custom look or behaviour.

---

*Artix OpenRC only – no systemd.*
