## Artix OpenRC Linux with LXQt Wayland desktop & labwc compositor
Artix base .iso March 2026
## 0. check disk names
```
lsblk
```
## 1. Wipe & Partition
```
wipefs -a /dev/sda
````
````
cfdisk /dev/sda
````
Partition type (gpt)

sda1 - 512M - EFI System

sda2 - 2M (min) - BIOS boot

sda3 - 128G - Linux filesystem   (root)

sda4 - rest - Linux filesystem   (home)
 
Write changes & quit

## 2. Format Partitions
```
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -f /dev/sda3
mkfs.btrfs -f /dev/sda4
```
## 3. Mount Partitions
```
mount /dev/sda3 /mnt
mkdir /mnt/home
mount /dev/sda4 /mnt/home
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```
## 4. Network configuration (ConnMan)
```
connmanctl
enable wifi
scan wifi
services
agent on
connect wifi_xxxxxxxxxx     # ← your network name from services
quit
```
## Optional – time sync
```
rc-service ntpd start
```
## Add rank best artix mirrors
```
pacman -Syy
pacman -S pacman-contrib
```
```
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
rankmirrors /etc/pacman.d/mirrorlist-artix > /etc/pacman.d/mirrorlist
```
## 5. Base system setup
```
basestrap /mnt base base-devel openrc elogind-openrc linux linux-headers linux-firmware sof-firmware intel-ucode intel-media-driver libva-utils nano
```
```
bash -c 'fstabgen -U /mnt > /mnt/etc/fstab'
```
## 6. Chroot system settings
```
artix-chroot /mnt
ln -sf /usr/share/zoneinfo/UK/London /etc/localtime
hwclock --systohc
nano /etc/locale.gen
```
Uncomment: required language / country
en_GB.UTF-8 UTF-8
```
locale-gen
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
echo "YOURHOSTNAME" > /etc/hostname
echo "hostname=\"YOURHOSTNAME\"" > /etc/conf.d/hostname
```
replace "YOURHOSTNAME"

## Add host & user details
```
nano /etc/hosts
```
127.0.1.1   artixhost.localdomain artixhost
## Add system password
```
passwd
```
## Add User & password
```
useradd -m -G wheel yourusername
passwd yourusername
```
replace "yourusername"
## uncomment:
%wheel ALL=(ALL:ALL) ALL
```
EDITOR=nano visudo
```
## 7. Install Desktop, tools, packages
```
pacman -Syu grub efibootmgr lxqt lxqt-wayland-session labwc xorg-xwayland qt6-wayland gvfs mesa mesa-utils vulkan-intel vulkan-tools fprintd networkmanager networkmanager-openrc network-manager-applet blueman bluez bluez-openrc bluez-utils alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber wireplumber-openrc pavucontrol-qt
```
## 8. GRUB (UEFI + legacy BIOS)
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix --removable
grub-install --target=i386-pc --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```
## 9. Services
```
rc-update add NetworkManager default
rc-update add bluetoothd default
rc-update add fprintd default
```
## 10. Finish

exit

umount -R /mnt

reboot

----------------------------------------------------------------------------------------------------------------------------------------------------------------------------

## Post-install – PipeWire autostart (run as root) su
```
mkdir -p /etc/xdg/autostart

cat > /etc/xdg/autostart/pipewire.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PipeWire
Comment=Multimedia server
Exec=/usr/bin/pipewire
Hidden=false
NoDisplay=true
X-GNOME-Autostart-enabled=true
EOF

cat > /etc/xdg/autostart/pipewire-pulse.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=PipeWire Pulse
Comment=PulseAudio replacement
Exec=/usr/bin/pipewire-pulse
Hidden=false
NoDisplay=true
X-GNOME-Autostart-enabled=true
EOF

cat > /etc/xdg/autostart/wireplumber.desktop << 'EOF'
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
Then log out / reboot → audio should work
