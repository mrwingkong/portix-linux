## Copy paste between host and virtual management such as gnome boxes
```
sudo pacman -S spice-vdagent-openrc
sudo rc-update add spice-vdagent default
sudo rc-service spice-vdagent start
```
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



## maybe required for file transfers
spice-webdavd

## Flutter Doctor / Dart requirements
[ Potentially install "unzip" for flutter install in vscode ]
```
sudo pacman -S clang cmake ninja mesa-utils
```
## Edit .bashrc
Add flutter chrome redirect to brave
Add flutter to path
```
export CHROME_EXECUTABLE=/opt/brave-bin/brave-browser
export PATH="$HOME/flutter/bin:$PATH"
