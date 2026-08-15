# portix-linux

Complete step-by-step guides to install **Artix Linux (OpenRC)** with an **LXQt + KWin Wayland** desktop. KDE Plasma will also works fine if fully installed. 

This project is focused on Wayland.  
If you need X11 instead, install `xorg-server` and `xorg-xinit` during the base install, then start the session with `startx`.  
For Wayland use `startlxqtwayland`.

**Features**
- Pure OpenRC (no systemd)
- KWin compositor
- Optional modular `.xzm` packages (inspired by Porteus Linux)
- Designed for portability — works great from a USB stick as a full installed system

Originally built and tested on the **Lenovo ThinkPad X1 2-in-1 Aura Edition (Intel Core Ultra)**.  
It has also run successfully on other 64-bit machines. Some unusual Wi-Fi cards (e.g. certain Broadcom chips in MacBooks) may need extra drivers.

### Support

If these guides helped you, feel free to buy me a coffee or a beer — your support is appreciated.

[![Buy Me a Coffee](https://img.shields.io/badge/Buy%20Me%20a%20Coffee-ffdd00?logo=buy-me-a-coffee&logoColor=black)](https://www.buymeacoffee.com/mrwingkong)

---

### Recommended install order

| File | Purpose |
|------|---------|
| [01-base-install.md](01-base-install.md) | Partitioning, base system, desktop packages, GRUB, basic services |
| [02-post-install.md](02-post-install.md) | Extra packages, yay, PowerDevil, UK keyboard, virtual keyboard, PipeWire |
| [03-swaync-notifications.md](03-swaync-notifications.md) | Disable LXQt notifications + set up swaync (needed before volume/brightness) |
| [04-biometrics.md](04-biometrics.md) | Fingerprint reader |
| [05-volume-osd.md](05-volume-osd.md) | Volume keys + OSD |
| [06-brightness-osd.md](06-brightness-osd.md) | Brightness keys + adaptive OSD |
| [07-wallpaper-orientation.md](07-wallpaper-orientation.md) | Landscape / portrait wallpapers for tablet mode |
| [08-misc.md](08-misc.md) | Extra tools (Android Studio, GNOME Boxes, bashrc tweaks, etc.) |

### Important notes

- All guides are pure **OpenRC** only.
- Commands are written for easy copy-paste.
- Replace `myname` with your real username and `myhostname` with your chosen hostname.
- LabWC is included in the base packages only so you can choose a compositor at first login. All detailed guides focus on KWin.
- Follow the files in order for the smoothest install.

**Start here → [01-base-install.md](01-base-install.md)**
