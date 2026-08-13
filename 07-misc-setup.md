# 07 – Misc Setup

Extra items that are useful but not required for a working desktop: GNOME Boxes + spice, .bashrc addons, Android Studio notes, Brave redirect, KVM group, desktop Exec fixes, etc.

---

## Prerequisites (already done in 01 / 02)

- User `myname` exists
- `yay` is installed
- Basic network and desktop are working

---

## 1. GNOME Boxes / virtual machines – spice-vdagent

Copy-paste and proper integration between host and guest:

```bash
sudo pacman -S --needed spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```

Optional (file transfers inside Boxes):

```bash
sudo pacman -S --needed spice-webdavd
```

---

## 2. KVM hardware acceleration group

```bash
sudo usermod -aG kvm myname
```

Log out and back in for the group to take effect.

---

## 3. .bashrc addons (Chrome → Brave redirect + Flutter path)

```bash
nano ~/.bashrc
```

Add at the end (adjust Flutter path if different):

```bash
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
```

Then:

```bash
source ~/.bashrc
```

---

## 4. Android Studio notes

- Install via AUR if desired: `yay -S android-studio`
- Or download the official tarball and place it under `~/Android/android-studio` (or similar).
- Make sure the `PATH` and any required environment variables are in `~/.bashrc` or `~/.config/environment.d/`.
- For Wayland: most recent versions work; if you hit rendering issues, try launching with `QT_QPA_PLATFORM=xcb` or the official Wayland flags.

No special OpenRC service is required.

---

## 5. Desktop Exec fixes (apps that need a terminal)

For shortcuts that fail because they expect a terminal or root privileges, edit the `.desktop` file (usually in `~/.local/share/applications/` or `/usr/share/applications/`).

Example pattern:

```
Exec=lxqt-sudo qterminal -e /home/myname/.local/bin/btop
```

- Remove any trailing `%F` / `%U` if it causes problems.
- Use `lxqt-sudo` when the command needs elevated rights.
- Use `qterminal -e` (or your preferred terminal) when a TTY is required.

---

## 6. Optional – Brave as default browser

If you installed `brave-bin` in post-install, set it as default in:

**LXQt Settings → Session Settings → Default Applications**  
or via `xdg-settings set default-web-browser brave-browser.desktop`

---

## 7. Quick reference – useful groups

After all guides you should typically be in:

```bash
groups
```

Expected (among others): `wheel`, `video`, `kvm` (and any others you added).

---

## Final check

After finishing all guides, a full reboot is recommended so every OpenRC service, environment file and autostart entry is cleanly loaded.

You should now have:

- Working LXQt + KWin Wayland
- PowerDevil power management
- UK keyboard
- Plasma virtual keyboard (tablet mode)
- Fingerprint login / sudo
- Volume + brightness OSDs via swaync
- Orientation-aware wallpapers
- spice-vdagent for GNOME Boxes
- Local scripts in `~/.local/bin`

---

*Artix OpenRC only – no systemd.*  
*Personal project inspired by Porteus-style modular thinking.*
