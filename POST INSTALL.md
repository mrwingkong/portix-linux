## Add for fastest mirrors
```
pacman -Syy
pacman-contrib
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
rankmirrors /etc/pacman.d/mirrorlist-artix > /etc/pacman.d/mirrorlist
```
## Packages for custom setup including display and system tools for a more fully functional desktop + extra's
```
libnotify wlr-randr kanshi swaybg libarchive libstatgrab git wget unzip xz btrfs-progs ntfs-3g exfatprogs xfsprogs e2fsprogs f2fs-tools dosfstools qt6-tools squashfs-tools sed mujs clang cmake ninja qemu-base libbsd
```
## AUR package installations - yay AUR helper
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
```
```
yay -S brave-bin
yay -S wdisplays-git
yay -S labwc-teaks-git
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
```
## Add flutter chrome redirect to brave / Add flutter to path
add to .bashrc
```
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
```
```
## Virtual management / KVM hardware acceleration user permission (NOT NEEDED FOR WAYLAND)
```
sudo usermod -aG kvm mrwingkong
```
## For apps not loading that require terminal

Change .desktop shortcut details (add lxqt sudo if root privalages needed, add terminal program -e)
remove the &F
```
Exec=lxqt-sudo qterminal -e /home/mrwingkong/.local/bin/btop
```
