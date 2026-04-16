## Copy paste between host and virtual management gnome boxes
```
sudo pacman -S spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```
## maybe required for file transfers

spice-webdavd

```
```
## Installing yay & brave browser
```
git clone https://aur.archlinux.org/yay.git
cd yay
makepkg -si
cd ~
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
