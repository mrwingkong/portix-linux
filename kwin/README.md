# KWin Path

Documentation for running LXQt under the **KWin** Wayland compositor.

KWin offers better HDR support and tighter integration with Plasma components (PowerDevil, kscreen, etc.).

### Guides in this folder

| File | Description |
|------|-------------|
| [full-osd-actions.md](full-osd-actions.md) | **Recommended** – complete volume + brightness OSD setup |
| [swaync-setup.md](swaync-setup.md) | SwayNC notification daemon only |
| [volume-osd-setup.md](volume-osd-setup.md) | Volume keys + OSD + ALSA levels |
| [brightness-osd-setup.md](brightness-osd-setup.md) | Adaptive brightness with true 0 % = off |
| [wallpaper-consistency.md](wallpaper-consistency.md) | Wallpaper consistency fix |

### Critical first steps

```bash
sudo pacman -S --needed sxhkd brightnessctl swaync libnotify pipewire pipewire-pulse wireplumber alsa-utils kscreen
```
sudo usermod -aG video $USER
```
Log out and back in after adding yourself to the video group.
Also disable PowerDevil brightness shortcuts:

Open System Settings → Shortcuts
Search for brightness
Set the four screen brightness actions to None

Before you start
Make sure you have completed the KWin base install from the base-install/ folder first.
