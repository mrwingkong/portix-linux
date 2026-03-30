## Copy paste between host and virtual management gnome boxes
```
sudo pacman -S spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```
## maybe required for file transfers
spice-webdavd

## Extra packages
```
sudo pacman -S  git
sudo pacman -S  wget
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
sudo pacman -S unzip clang cmake ninja mesa-utils qemu-base libbsd

```
```
sudo usermod -aG kvm $USER
echo -e "kvm\nkvm_intel\nkvm_amd" | sudo tee /etc/modules-load.d/kvm.conf
```
## Edit .bashrc
Add flutter chrome redirect to brave
Add flutter to path
```
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
