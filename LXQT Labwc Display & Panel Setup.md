# LXQt + Labwc: Kanshi Display Profiles + LXQt Panel Restart

**Purpose**: Automatically apply your exact monitor layouts (including 270° portrait rotation) when you plug/unplug displays. On every change it restarts both the LXQt panel **and** the wallpaper setter so you never get solid-color screens or squashed/misplaced content on the laptop display after reconnecting the portrait monitor.

Kanshi handles the layout switching. The custom restart script + delayed autostart + hidden LXQt panel entry fixes the common Wayland hotplug races.

---

## 1. Install Core Packages

```bash
sudo pacman -S kanshi wlr-randr wdisplays swaybg
```

- `kanshi` — dynamic output profiles (the core of persistent display memory)
- `wlr-randr` — inspect current outputs/modes from CLI
- `wdisplays` — GUI to arrange monitors + export kanshi config (strongly recommended)
- `swaybg` — sets wallpaper on all outputs; we now restart it on every hotplug to prevent solid-color screens

---

## 2. Panel + Wallpaper Restart Script (triggered by kanshi on every hotplug)

This is the key fix for your issue.

When you reconnect the portrait monitor, kanshi applies the new layout, but the wallpaper (and sometimes the panel) needs to be explicitly repainted on the newly enabled/rotated output. Without this, you get solid color on the external screen and visual corruption (squashed/misplaced content) on the laptop display.

```bash
mkdir -p ~/.local/bin

cat > ~/.local/bin/restart-lxqt-panel.sh << 'EOF'
#!/bin/sh
echo "[$(date '+%F %T')] kanshi profile applied - restarting panel + wallpaper" >> ~/.kanshi-panel.log

# Kill existing instances
pkill -x lxqt-panel 2>/dev/null || true
pkill -x swaybg 2>/dev/null || true

# Give outputs time to fully enable/rotate after kanshi change (important for 270° portrait)
sleep 2

# Re-apply wallpaper to ALL current outputs (fixes solid color on reconnected portrait screen)
swaybg -i /usr/share/lxqt/wallpapers/origami-dark-labwc.png >/dev/null 2>&1 &

# Restart panel (prevents duplicates and fixes positioning after layout change)
lxqt-panel &

echo "[$(date '+%F %T')] Panel + wallpaper restarted cleanly" >> ~/.kanshi-panel.log
EOF

chmod +x ~/.local/bin/restart-lxqt-panel.sh
```

**Tip**: If you use a different wallpaper image, change the path in the script (and in autostart below).

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

The initial `sleep` prevents race conditions on login.  
`swaybg` is started here once at boot; the restart script (section 2) re-runs it on every hotplug so new/rotated outputs always get the wallpaper.

```bash
mkdir -p ~/.config/labwc

cat > ~/.config/labwc/autostart << 'EOF'
# Kill any previous swaybg and set wallpaper on all outputs
pkill -x swaybg 2>/dev/null || true
swaybg -i /usr/share/lxqt/wallpapers/origami-dark-labwc.png >/dev/null 2>&1 &

# Wayland environment vars
dbus-update-activation-environment --systemd DISPLAY WAYLAND_DISPLAY >/dev/null 2>&1 &

# Delayed kanshi start (tune 4-8s if you still see duplicate panels on login)
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

After plugging/unplugging a monitor (especially the portrait one):

```bash
tail -n 30 ~/.kanshi-panel.log
```

You should see clean "Panel + wallpaper restarted cleanly" entries. The external portrait screen should now have the wallpaper (not solid color), and the laptop screen should show a single clean image without squashed/misplaced content from the other monitor.

**If you still see solid color on the reconnected screen or squashing on the laptop:**
- Increase `sleep 2` to `sleep 3` in the restart script.
- Make sure the exact `mode`, `position`, `transform`, and `scale` lines in your kanshi profiles match what `wlr-randr` reports when that monitor is connected.
- Re-export fresh profiles with `wdisplays` after a clean reconnect.
- Check `journalctl --user -u kanshi` or run `kanshi` manually in a terminal for errors.

---

**That's it.**  
No idle/sleep display management, no unnecessary complexity. Just reliable display memory + panel that survives hotplug events.
