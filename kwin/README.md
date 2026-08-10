# Modular LXQt + KWin Wayland OSD Guides

These guides are the split version of the full working configuration.

## Order of application (recommended)

1. **01-swaync-setup.md**  
   Notification daemon required by the OSDs.

2. **02-volume-osd-setup.md**  
   Volume keys + OSD + ALSA hardware levels + USB-C support.

3. **03-brightness-osd-setup.md**  
   Adaptive brightness with true 0 % = off in both SDR and HDR.

You can also use the single combined guide:

- `lxqt-wayland-kwin-swaync-osd-actions.md`

## Critical shared requirements

```bash
sudo pacman -S --needed sxhkd brightnessctl swaync libnotify \
  pipewire pipewire-pulse wireplumber alsa-utils kscreen

sudo usermod -aG video $USER
# then log out and back in
```

## After any of the guides

Always start the services once for the current session (or just log out/in):

```bash
pkill sxhkd swaync 2>/dev/null
swaync &
sxhkd &
```

## Disable PowerDevil brightness shortcuts

System Settings → Shortcuts → search “brightness” → set the four screen brightness actions to **None**.
