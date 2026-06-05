## Additional packages / installations
```
sudo pacman -S libva libva-utils intel-media-driver xf86-video-nouveau
```
## Copy paste between host and virtual management specifically 'gnome boxes'
```
sudo pacman -S spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```
Possible requirement for file transfers
```
sudo pacman -S spice-webdavd
```
## 
## For apps not loading that require terminal

Change .desktop shortcut details (add lxqt sudo if root privalages needed, add terminal program -e)
remove the &F
```
Exec=lxqt-sudo qterminal -e /home/mrwingkong/.local/bin/btop
```
## AUR package installtions - yay AUR helper
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
```
example for brave web browser
```
yay -Sy brave-bin
```
## Android Studio / VSCode / Flutter Dart App Development
```
sudo pacman -S clang cmake ninja qemu-base libbsd

```
## Virtual management / KVM hardware acceleration user permission
```
sudo usermod -aG kvm mrwingkong
```
## Edit .bashrc
Add flutter chrome redirect to brave
Add flutter to path
```
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
