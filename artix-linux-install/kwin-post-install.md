## Add for fastest mirrors

```
pacman -Syy
pacman-contrib
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
rankmirrors /etc/pacman.d/mirrorlist-artix > /etc/pacman.d/mirrorlist
```

## Packages for custom setup including display and system tools for a more fully functional desktop + extra's

```
fprintd qt6-tools xdg-desktop-portal xdg-desktop-portal-kde swaybg swaync plasma-keyboard libnotify brightnessctl sxhkd iio-sensor-proxy python-pyqt6 libarchive libstatgrab git wget unzip xz btrfs-progs ntfs-3g exfatprogs xfsprogs e2fsprogs f2fs-tools squashfs-tools sed mujs clang cmake ninja qemu-base libbsd
```

## AUR package installations - yay AUR helper

```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
yay -S brave-bin
```

## Copy paste between host and virtual management specifically 'gnome boxes'

```
sudo pacman -S spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```

Possible requirement for file transfers in 'gnome boxes'

```
sudo pacman -S spice-webdavd
```

## Add chrome browser redirect to brave browser / Add Flutter to system path

Add to `.bashrc`:

```
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
```

## Virtual management / KVM hardware acceleration user permission (NOT NEEDED FOR WAYLAND)

```
sudo usermod -aG kvm mrwingkong
```

## For apps not loading that require terminal

Change `.desktop` shortcut details (add `lxqt-sudo` if root privileges needed, add terminal program `-e`).  
Remove the `&F`.

```
Exec=lxqt-sudo qterminal -e /home/mrwingkong/.local/bin/btop
```

## Power Management (PowerDevil) – LXQt + KWin

PowerDevil handles idle dimming, screen off, suspend, lid close, and battery events under KWin.

### Start at login

```
mkdir -p ~/.config/autostart

cat > ~/.config/autostart/powerdevil.desktop << 'EOT'
[Desktop Entry]
Type=Application
Name=PowerDevil
Exec=/usr/lib/org_kde_powerdevil
X-GNOME-Autostart-enabled=true
EOT
```

Start it now (if not already running):

```
/usr/lib/org_kde_powerdevil &
```

### Open settings

```
systemsettings
```

Go to **Energy Saving** / **Power Management**.

Useful options:

| Setting | Notes |
|---------|--------|
| Dim screen after | 5–10 min, or disable if it fights custom brightness scripts |
| Turn off screen after | 10–15 min |
| Suspend session after | As preferred (or Never on AC) |
| Lid closed | Suspend / Lock / Turn off screen |

### Disable PowerDevil brightness keys

Custom `brightness-osd.sh` owns the brightness keys. Disable PowerDevil’s copies so they do not conflict:

1. **System Settings → Shortcuts**
2. Search **brightness**
3. Set to **None**:
   - Increase Screen Brightness
   - Decrease Screen Brightness
   - Increase Screen Brightness by 1%
   - Decrease Screen Brightness by 1%

PowerDevil still manages idle, lock, lid, and battery.

### Verify

```
ps aux | grep -i powerdevil | grep -v grep
```
