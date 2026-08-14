# portix-linux
**(KWin & pman module intergration project)**

**Artix OpenRC Linux with LXQt Wayland using KWin compositor**

Custom Artix Linux, inspired by Porteus Linux and its derivatives (especially the modular `.xzm` package approach).

Originally developed and tested on the **Lenovo ThinkPad X1 2-in-1 Aura Edition (Intel Core Ultra)**.

This repository provides clean, copy-paste Markdown guides for a fully working LXQt + KWin Wayland desktop on pure **OpenRC** (no systemd). LabWC is included in the base package list only so the LXQt session selector can offer a compositor choice on first boot; all detailed guides focus on the KWin path.

### Recommended install order

| File | Purpose |
|------|---------|
| [01-base-install.md](01-base-install.md) | Disk partitioning, base system, desktop packages (includes both KWin + LabWC), GRUB, basic OpenRC services |
| [02-post-install.md](02-post-install.md) | After first reboot – mirrors, extra packages, yay, PowerDevil, UK keyboard, Plasma virtual keyboard, audio reset, PipeWire autostarts |
| [03-swaync-notifications.md](03-swaync-notifications.md) | Disable LXQt notification daemon + install and autostart swaync (required before volume/brightness OSD) |
| [04-biometrics.md](04-biometrics.md) | Fingerprint reader (fprintd) + PAM for login/sudo/polkit |
| [05-volume-osd.md](05-volume-osd.md) | Volume keys + swaync OSD + ALSA hardware levels |
| [06-brightness-osd.md](06-brightness-osd.md) | Adaptive brightness (true 0 % = off) + HDR/SDR memory + swaync OSD |
| [07-wallpaper-orientation.md](07-wallpaper-orientation.md) | Orientation-aware wallpapers with swaybg (landscape ↔ portrait) |
| [08-misc.md](08-misc.md) | Android Studio notes, .bashrc addons, GNOME Boxes + spice-vdagent, Brave, KVM group, desktop Exec fixes, etc. |

### Key principles

- Pure **OpenRC** only (`rc-update`, `rc-service`, custom `/etc/init.d/` where needed).
- All guides use the same simple formatting – numbered steps and ready-to-copy code blocks.
- Username placeholder is the literal word **myname** everywhere (no quotes, no brackets).
- Hostname placeholder is **myhostname**.
- Designed so you can follow the numbered files in order on a fresh install with minimal friction.

### Notes

- Extra packages and scripts are tailored to the original hardware (Intel Arc, SOF audio, fingerprint, tablet mode / orientation sensor).
- The modular `.xzm` workflow (Porteus-style) lives in the older module-related files of the original repository if you still need it; this cleaned set focuses on the KWin desktop path.

Start with **01-base-install.md**.
