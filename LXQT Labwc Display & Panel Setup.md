## 1. Install Required Packages 
On a fresh updated system, install these:
```
sudo pacman -S

kanshi \ wlr-randr \ wdisplays \ swaybg \ swayidle \ wlopm
```
## 2. Create the Panel Restart Wrapper Script
```
mkdir -p ~/.local/bin
```
```
cat > ~/.local/bin/restart-lxqt-panel.sh << 'EOF'
#!/bin/sh
echo "=== PANEL RESTART TRIGGERED at $(date) ===" >> ~/.kanshi-panel.log


pkill -x lxqt-panel 2>/dev/null || true
pkill -9 -x lxqt-panel 2>/dev/null || true
sleep 2.0

lxqt-panel &

echo "=== PANEL RESTARTED at $(date) ===" >> ~/.kanshi-panel.log
EOF
```
```
chmod +x ~/.local/bin/restart-lxqt-panel.sh
```
## 3. Create the Kanshi Configuration (with 4 profiles)
```
mkdir -p ~/.config/kanshi
```
```
cat > ~/.config/kanshi/config << 'EOF'
# === FULL SETUP (your normal layout) ===
profile {
    output DP-2 {
        mode 1920x1080@60.000000
        position 0,0
        transform 270
        scale 1.0
    }
    output DP-1 {
        mode 2560x1440@59.951000
        position 1080,480
        transform normal
        scale 1.0
    }
    output eDP-1 {
        mode 2880x1800@60.000000
        position 3640,795
        transform normal
        scale 1.601562
    }
    exec ~/.local/bin/restart-lxqt-panel.sh
}

# === eDP LAPTOP ONLY ===
profile {
    output eDP-1 {
        mode 2880x1800@60.000000
        position 0,0
        transform normal
        scale 1.601562
    }
    exec ~/.local/bin/restart-lxqt-panel.sh
}

# === DP-1 (middle) + eDP laptop ===
profile {
    output DP-1 {
        mode 2560x1440@59.951000
        position 0,0
        transform normal
        scale 1.0
    }
    output eDP-1 {
        mode 2880x1800@60.000000
        position 2560,0
        transform normal
        scale 1.601562
    }
    exec ~/.local/bin/restart-lxqt-panel.sh
}

# === DP-2 (portrait) + eDP laptop ===
profile {
    output DP-2 {
        mode 1920x1080@60.000000
        position 0,0
        transform 270
        scale 1.0
    }
    output eDP-1 {
        mode 2880x1800@60.000000
        position 1080,0
        transform normal
        scale 1.601562
    }
    exec ~/.local/bin/restart-lxqt-panel.sh
}
EOF
```
## 4. Create the Autostart File
```
cat > ~/.config/labwc/autostart << 'EOF'
# Background
swaybg -i /usr/share/lxqt/wallpapers/origami-dark-labwc.png >/dev/null 2>&1 &

# GTK environment
dbus-update-activation-environment --systemd DISPLAY WAYLAND_DISPLAY >/dev/null 2>&1 &

# === PERMANENT MONITOR LAYOUT (kanshi) ===
# Delay helps prevent duplicate panels on reboot
sleep 6 && kanshi >/dev/null 2>&1 &

# Idle display off (optional but recommended)
swayidle -w timeout 300 "wlopm --off *" resume "wlopm --on *" >/dev/null 2>&1 &
EOF
```
## 5. Prevent LXQt from Auto-Starting the Panel
```
mkdir -p ~/.config/autostart
```
```
cat > ~/.config/autostart/lxqt-panel.desktop << 'EOF'
[Desktop Entry]
Type=Application
Exec=lxqt-panel
Hidden=true
X-LXQt-Module=true
EOF
```
## 6. Apply Everything
```
pkill -x kanshi 2>/dev/null || true
pkill -x lxqt-panel 2>/dev/null || true

kanshi >/dev/null 2>&1 &
lxqt-panel &
```
## 7. Final Test

Log out and log back in (or reboot).
After login you should have one panel and your monitors in the correct layout (including 270° rotation on DP-2).
Unplug/replug monitors and check that the panel restarts.

Check the log after a hotplug:

cat ~/.kanshi-panel.log | tail -n 10
