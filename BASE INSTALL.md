# Artix Linux OpenRC with LXQt desktop enviorment & xzm module support packages

(Minimal base Artix install iso as of March 2026)

Separate BTRFS root & home, hybrid UEFI+BIOS GRUB, SDDM login, LXQt desktop enviorment + additional tools & packages

## 1. Wipe & Partition
```
wipefs -a /dev/sda
dd if=/dev/zero of=/dev/sda bs=1M count=10 status=progress conv=fsync
sync
````
````
cfdisk /dev/sda
````
Partition type (gpt)

sda1 - 512M - EFI System

sda2 - 2M - BIOS boot

sda3 - 128G - Linux filesystem   (root)

sda4 - rest - Linux filesystem   (home)
 
Write changes & quit
```
wipefs --all --force /dev/sda2
dd if=/dev/zero of=/dev/sda2 bs=1M count=2 status=progress conv=fsync
sync
lsblk -f /dev/sda   # confirm sda2 has no filesystem
```
## 2. Format Partitions
```
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -f /dev/sda3
mkfs.btrfs -f /dev/sda4
```
Fix "Not automatically fixing this" label on EFI partition (clears dirty bit and repairs filesystem cleanly)
auto-repair; if "device busy", umount first then remount after
```
fsck.vfat -a /dev/sda1
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
## 5. Base system setup
```
basestrap /mnt base base-devel openrc elogind-openrc linux linux-headers linux-firmware intel-ucode sof-firmware nano
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
pacman -Syu grub efibootmgr lxqt sddm sddm-openrc mesa mesa-utils vulkan-intel vulkan-tools networkmanager networkmanager-openrc network-manager-applet blueman bluez bluez-openrc bluez-utils alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber wireplumber-openrc pavucontrol-qt xfwm4 picom nemo git wget squashfs-tools sed mujs pacman-contrib libarchive libstatgrab unzip xz btrfs-progs ntfs-3g exfatprogs xfsprogs e2fsprogs f2fs-tools dosfstools spice-vdagent-openrc
```
## 8. GRUB (UEFI + legacy BIOS)
```
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix --removable
grub-install --target=i386-pc --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```
## 9. Services
```
rc-update add sddm default
rc-update add NetworkManager default
rc-update add bluetoothd default
rc-update add spice-vdagent default
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
