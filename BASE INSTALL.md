# Artix Linux OpenRC + LXQt Installation Guide

Clean install with separate BTRFS root + home, hybrid UEFI+BIOS GRUB, PipeWire + WirePlumber, SDDM + LXQt.

**Assumes Artix base or LXQt live ISO (March 2026 workflow)**

## 1. Wipe & Partition

wipefs -a /dev/sda

dd if=/dev/zero of=/dev/sda bs=1M count=10 status=progress conv=fsync
sync

cfdisk /dev/sda
# Inside cfdisk – example layout:
 sda1    512M   EFI System
 
 sda2      2M   BIOS boot
 
 sda3    128G   Linux filesystem   (root)
 
 sda4    rest   Linux filesystem   (home)
 
Write changes and quit

wipefs --all --force /dev/sda2

dd if=/dev/zero of=/dev/sda2 bs=1M count=2 status=progress conv=fsync

sync

lsblk -f /dev/sda   # confirm sda2 has no filesystem

## 2. Format

mkfs.fat -F32 /dev/sda1

mkfs.btrfs -f /dev/sda3

mkfs.btrfs -f /dev/sda4

fsck.vfat -a /dev/sda1

## 3. Mount

mount /dev/sda3 /mnt

mkdir /mnt/home

mount /dev/sda4 /mnt/home

mkdir -p /mnt/boot

mount /dev/sda1 /mnt/boot

## 4. Network (ConnMan)

connmanctl
# inside connmanctl:
enable wifi

scan wifi

services

agent on

connect wifi_xxxxxxxxxx     # ← your network name from services

quit

# Optional – time sync
rc-service ntpd start

## 5. Base system

basestrap /mnt base base-devel openrc elogind-openrc linux linux-headers linux-firmware intel-ucode sof-firmware nano

fstabgen -U /mnt > /mnt/etc/fstab

## 6. Chroot & basic setup

artix-chroot /mnt

ln -sf /usr/share/zoneinfo/UK/London /etc/localtime

hwclock --systohc

nano /etc/locale.gen
# uncomment:
en_GB.UTF-8 UTF-8
    --------------

locale-gen

echo "LANG=en_GB.UTF-8" > /etc/locale.conf

echo "artix-lxqt" > /etc/hostname

echo 'hostname="artix-lxqt"' > /etc/conf.d/hostname

nano /etc/hosts
# Add:
127.0.0.1       localhost

::1             localhost

127.0.1.1       artix-lxqt.localdomain  artix-lxqt

passwd

useradd -m -G wheel yourusername
passwd yourusername

EDITOR=nano visudo
# uncomment:
%wheel ALL=(ALL:ALL) ALL
    ---------------------
## 7. Desktop & packages

pacman -Syu --needed grub efibootmgr lxqt sddm sddm-openrc mesa networkmanager networkmanager-openrc network-manager-applet alsa-utils squashfs-tools pacman-contrib sed xz libarchive libstatgrab pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber wireplumber-openrc pavucontrol-qt xfwm4 picom nemo btrfs-progs dosfstools exfatprogs ntfs-3g xfsprogs e2fsprogs f2fs-tools

## 8. GRUB (UEFI + legacy BIOS)

grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix --removable
grub-install --target=i386-pc --recheck /dev/sda

grub-mkconfig -o /boot/grub/grub.cfg

## 9. Services

rc-update add sddm default

rc-update add NetworkManager default

## 10. Finish

exit
umount -R /mnt
reboot


====================================================================================

## Post-install – PipeWire autostart (run as root)

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

# Then log out / reboot → audio should work
