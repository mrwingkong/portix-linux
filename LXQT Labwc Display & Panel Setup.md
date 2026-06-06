# LXQt + Labwc: Kanshi Display Profiles + LXQt Panel Restart

**Purpose**: Automatically apply your preferred monitor layouts (resolution, position, rotation, scale) when plugging/unplugging displays. Restart the LXQt panel on every change so it doesn't disappear or appear duplicated/wrongly positioned.

Kanshi watches for display changes and applies the matching profile + runs your restart script.  
A startup delay + hidden LXQt autostart prevents race conditions that cause duplicate or missing panels on login/reboot.

---

## 1. Install Core Packages

```bash
sudo pacman -S kanshi wlr-randr wdisplays
```

- `kanshi` — the heart (dynamic output profiles daemon)
- `wlr-randr` — CLI tool to inspect current outputs/modes
- `wdisplays` — GUI to visually arrange displays and generate kanshi snippets (highly recommended)

(Optional) If you want a simple wallpaper via labwc:
```bash
sudo pacman -S swaybg
```

---

## 2. Panel Restart Script (triggered by kanshi)

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/restart-lxqt-panel.sh << 'EOF'
#!/bin/sh
echo "[$(date '+%F %T')] Panel restart triggered" >> ~/.kanshi-panel.log

pkill -x lxqt-panel 2>/dev/null || true
sleep 1.5
lxqt-panel &

echo "[$(date '+%F %T')] Panel restarted" >> ~/.kanshi-panel.log
EOF

chmod +x ~/.local/bin/restart-lxqt-panel.sh
```

---

## 3. Kanshi Configuration (your display profiles)

**Important**: The example below is hardware-specific. **You must replace it with your own layouts.**

How to create your profiles easily:
1. Connect all your monitors the way you want.
2. Run `wdisplays` → drag/rotate/scale to taste → click the kanshi export button (or copy manually).
3. Add `exec ~/.local/bin/restart-lxqt-panel.sh` at the end of **every** profile block.
4. Order profiles from most outputs to fewest (kanshi uses the first matching profile).

```bash
mkdir -p ~/.config/kanshi

cat > ~/.config/kanshi/config << 'YOUR_PROFILES_HERE'
# === EXAMPLE: Laptop + 2 external monitors (replace with yours) ===
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

# Laptop screen only
profile {
    output eDP-1 {
        mode 2880x1800@60.000000
        position 0,0
        transform normal
        scale 1.601562
    }
    exec ~/.local/bin/restart-lxqt-panel.sh
}

# DP-1 + laptop only
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

# DP-2 (rotated) + laptop only
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
YOUR_PROFILES_HERE
```

---

## 4. Labwc Autostart (delayed kanshi start)

The `sleep` prevents kanshi + panel from starting before the LXQt session is fully ready (common cause of duplicate/missing panels).

```bash
mkdir -p ~/.config/labwc

cat > ~/.config/labwc/autostart << 'EOF'
# Optional wallpaper
swaybg -i /usr/share/lxqt/wallpapers/origami-dark-labwc.png >/dev/null 2>&1 &

# Wayland environment
dbus-update-activation-environment --systemd DISPLAY WAYLAND_DISPLAY >/dev/null 2>&1 &

# Start kanshi after a short delay (adjust 4-8s if you still get duplicate panels)
sleep 6 && kanshi >/dev/null 2>&1 &
EOF
```

---

## 5. Disable LXQt's Built-in Panel Autostart

This stops LXQt from launching `lxqt-panel` on its own. The panel will only be started by kanshi (after displays are configured).

```bash
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/lxqt-panel.desktop << 'EOF'
[Desktop Entry]
Type=Application
Exec=lxqt-panel
Hidden=true
X-LXQt-Module=true
EOF
```

---

## 6. Apply Changes (one-time)

```bash
pkill -x kanshi 2>/dev/null || true
pkill -x lxqt-panel 2>/dev/null || true

kanshi >/dev/null 2>&1 &
lxqt-panel &
```

Log out and back in (or reboot) to test the full flow.

---

## 7. Verify & Debug Hotplug

After plugging/unplugging a monitor:

```bash
tail -n 30 ~/.kanshi-panel.log
```

You should see restart entries and your layout should be correct with exactly one panel.

If the panel is still missing after hotplug, increase the `sleep` in the restart script or in autostart.

---

**That's it.**  
No idle/sleep display management, no unnecessary complexity. Just reliable display memory + panel that survives hotplug events.

cat ~/.kanshi-panel.log | tail -n 10
