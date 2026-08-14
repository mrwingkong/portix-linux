# 02 – Post Install

Run these steps **after the first reboot** from 01-base-install.md, logged in as **myname**.

This file finishes the majority of the system: extra packages, yay, PowerDevil, UK keyboard, Plasma virtual keyboard, PipeWire autostarts, and the critical audio hardware reset script.

---

## 1. Fastest mirrors (again)

```bash
sudo pacman -Syy
sudo pacman -S --needed pacman-contrib
```

```bash
sudo cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
sudo rankmirrors /etc/pacman.d/mirrorlist-artix | sudo tee /etc/pacman.d/mirrorlist
```

---

## 2. Extra packages for a fully functional desktop

```bash
sudo pacman -S --needed \
  fprintd qt6-tools xdg-desktop-portal xdg-desktop-portal-kde \
  swaybg swaync plasma-keyboard libnotify brightnessctl sxhkd \
  iio-sensor-proxy python-pyqt6 \
  libarchive libstatgrab git wget unzip xz \
  btrfs-progs ntfs-3g exfatprogs xfsprogs e2fsprogs f2fs-tools dosfstools \
  squashfs-tools sed mujs clang cmake ninja qemu-base libbsd
```

---

## 3. yay AUR helper + Brave

```bash
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
```

```bash
yay -S brave-bin
```

---

## 4. PipeWire autostart (system-wide)

```bash
sudo mkdir -p /etc/xdg/autostart
```

```bash
sudo tee /etc/xdg/autostart/pipewire.desktop > /dev/null << 'EOF'
[Desktop Entry]
Type=Application
Name=PipeWire
Comment=Multimedia server
Exec=/usr/bin/pipewire
Hidden=false
NoDisplay=true
X-GNOME-Autostart-enabled=true
EOF
```

```bash
sudo tee /etc/xdg/autostart/pipewire-pulse.desktop > /dev/null << 'EOF'
[Desktop Entry]
Type=Application
Name=PipeWire Pulse
Comment=PulseAudio replacement
Exec=/usr/bin/pipewire-pulse
Hidden=false
NoDisplay=true
X-GNOME-Autostart-enabled=true
EOF
```

```bash
sudo tee /etc/xdg/autostart/wireplumber.desktop > /dev/null << 'EOF'
[Desktop Entry]
Type=Application
Name=WirePlumber
Comment=Session/policy manager for PipeWire
Exec=/usr/bin/wireplumber
Hidden=false
NoDisplay=true
X-GNOME-Autostart-enabled=true
EOF
```

---

## 5. PowerDevil (power management)

PowerDevil handles idle dimming, screen off, suspend, lid close and battery events under KWin.

### 5.1 Create the autostart file

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/powerdevil.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PowerDevil
Exec=/usr/lib/org_kde_powerdevil
X-GNOME-Autostart-enabled=true
EOF
```

### 5.2 Start PowerDevil for the current session

Run this exact command:

```bash
/usr/lib/org_kde_powerdevil &
```

Check it is running:

```bash
ps aux | grep -i powerdevil | grep -v grep
```

You should see a process for `org_kde_powerdevil`.

If it is not running, start it in the foreground (this is what fixed it during testing):

```bash
/usr/lib/org_kde_powerdevil
```

Leave it running, or press Ctrl+C and then start it again with `&`.

### 5.3 Log out and log back in

After creating the autostart file, log out and log back in so the session starts PowerDevil cleanly.

Confirm after login:

```bash
ps aux | grep -i powerdevil | grep -v grep
```

### 5.4 Open settings

```bash
systemsettings
```

Go to **Energy Saving / Power Management** and set your preferred values (dim, turn off screen, suspend, lid closed behaviour).

### 5.5 Disable PowerDevil brightness shortcuts (CRITICAL)

Custom brightness scripts (guide 05) will own the brightness keys.  
Disable PowerDevil’s copies so they do not conflict:

1. Open **System Settings → Shortcuts**
2. Search for `brightness`
3. Set these four actions to **None**:
   - Increase Screen Brightness
   - Decrease Screen Brightness
   - Increase Screen Brightness by 1%
   - Decrease Screen Brightness by 1%

PowerDevil still manages idle, lock, lid and battery – only the brightness keys are released.

---

## 6. UK Keyboard Layout

Pure OpenRC method (no localectl / systemd).

### Console / TTY

```bash
sudo nano /etc/vconsole.conf
```

Set the file to:

```
KEYMAP=uk
```

### Wayland + XWayland

```bash
mkdir -p ~/.config/environment.d
```

```bash
cat > ~/.config/environment.d/00-keyboard.conf << 'EOF'
XKB_DEFAULT_LAYOUT=gb
XKB_DEFAULT_MODEL=pc105
EOF
```

### KWin layout

```bash
cat > ~/.config/kxkbrc << 'EOF'
[Layout]
LayoutList=gb
Use=true
EOF
```

Log out and back in (or reboot) for the graphical session to pick up the layout.

Test: Shift+2 should produce `"` and Shift+' should produce `@`.

---

## 7. Plasma Virtual Keyboard (tablet mode)

```bash
sudo pacman -S --needed plasma-keyboard
```

Point KWin at it:

```bash
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod "/usr/share/applications/org.kde.plasma.keyboard.desktop"
```

Verify the desktop file exists:

```bash
ls /usr/share/applications/*plasma*keyboard*
```

Log out and back in.

In tablet mode, tap any text field – the Plasma Keyboard should appear.

Optional (only if it never appears on touch):

```bash
echo 'KWIN_IM_SHOW_ALWAYS=1' >> ~/.config/environment.d/plasma-keyboard.conf
```

---

## 8. Audio hardware reset script (Intel SOF)

This forces the Speaker and Headphone hardware channels to 100 % so software volume control works correctly.  
It is placed here (after all packages) so it can be used immediately and is also autostarted by the volume OSD guide.

```bash
mkdir -p ~/.local/bin
```

```bash
cat > ~/.local/bin/volume-reset.sh << 'EOF'
#!/bin/bash
sleep 5
amixer -c 0 -q sset Speaker 100% unmute
amixer -c 0 -q sset Headphone 100% unmute
wpctl set-volume @DEFAULT_AUDIO_SINK@ 0% 2>/dev/null
wpctl set-mute @DEFAULT_AUDIO_SINK@ 0 2>/dev/null
EOF
chmod +x ~/.local/bin/volume-reset.sh
```

Run it once now:

```bash
~/.local/bin/volume-reset.sh
```

---

## 9. Set KWin as default compositor (optional but recommended)

```bash
mkdir -p ~/.config/lxqt
```

```bash
cat > ~/.config/lxqt/session.conf << 'EOF'
[General]
compositor=kwin_wayland
EOF
```

---

## 10. PATH for local scripts

```bash
echo 'export PATH="$HOME/.local/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
```

---

## Next steps

Continue with the specialised guides in order:

- **03-swaync-notifications.md** – disable LXQt notifications + setup swaync
- **04-biometrics.md** – fingerprint
- **05-volume-osd.md** – volume keys + OSD
- **06-brightness-osd.md** – brightness keys + OSD
- **07-wallpaper-orientation.md** – tablet mode wallpapers
- **08-misc.md** – remaining extras

A full logout/login (or reboot) after this post-install is recommended so all environment files and autostart entries take effect.

---

*Artix OpenRC only – no systemd.*
