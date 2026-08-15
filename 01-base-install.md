# 01 – Base Install

Artix OpenRC + LXQt Wayland (KWin focus)  
Target hardware: Lenovo ThinkPad X1 2-in-1 Aura Edition (Core Ultra)

Use a recent Artix base OpenRC ISO.  
All commands assume you are running as root from the live environment unless noted.

---

## 0. Check disk names

```bash
lsblk
```

Adjust `/dev/sda` in the following steps if your disk is different (e.g. `/dev/nvme0n1`).

---

## 1. Wipe & Partition

```bash
wipefs -a /dev/sda
```

```bash
cfdisk /dev/sda
```

Partition type: **gpt**

| Partition | Size     | Type              |
|-----------|----------|-------------------|
| sda1      | 512M     | EFI System        |
| sda2      | 2M (min) | BIOS boot         |
| sda3      | 128G     | Linux filesystem (root) |
| sda4      | rest     | Linux filesystem (home) |

Write changes & quit.

---

## 2. Format Partitions

```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -f /dev/sda3
mkfs.btrfs -f /dev/sda4
```

---

## 3. Mount Partitions

```bash
mount /dev/sda3 /mnt
mkdir /mnt/home
mount /dev/sda4 /mnt/home
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

---

## 4. Network (ConnMan – live environment)

```bash
connmanctl
```

Inside connmanctl:

```
enable wifi
scan wifi
services
agent on
connect wifi_xxxxxxxxxx     # replace with the service name shown
quit
```

Optional time sync:

```bash
rc-service ntpd start
```

---

## 5. Rank Artix mirrors

```bash
pacman -Syy
pacman -S pacman-contrib
```

```bash
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
rankmirrors /etc/pacman.d/mirrorlist-artix > /etc/pacman.d/mirrorlist
```

---

## 6. Base system (basestrap)

```bash
basestrap /mnt base base-devel openrc elogind-openrc linux linux-headers linux-firmware sof-firmware intel-ucode intel-media-driver libva-utils nano
```

```bash
bash -c 'fstabgen -U /mnt > /mnt/etc/fstab'
```

---

## 7. Chroot

```bash
artix-chroot /mnt
```

Timezone (UK example):

```bash
ln -sf /usr/share/zoneinfo/Europe/London /etc/localtime
hwclock --systohc
```

Locale:

```bash
nano /etc/locale.gen
```

Uncomment:

```
en_GB.UTF-8 UTF-8
```

```bash
locale-gen
echo "LANG=en_GB.UTF-8" > /etc/locale.conf
```

Hostname:

```bash
echo "myhostname" > /etc/hostname
echo 'hostname="myhostname"' > /etc/conf.d/hostname
```

Hosts file:

```bash
nano /etc/hosts
```

Add / adjust:

```
127.0.0.1   localhost
::1         localhost
127.0.1.1   myhostname.localdomain myhostname
```

Root password:

```bash
passwd
```

Create user:

```bash
useradd -m -G wheel myname
passwd myname
```

Sudo for wheel:

```bash
EDITOR=nano visudo
```

Uncomment the line:

```
%wheel ALL=(ALL:ALL) ALL
```

---

## 8. Desktop & core packages

LabWC is included **only** so the LXQt session selector can offer a compositor choice on first boot. All later guides focus on KWin.

```bash
pacman -Syu grub efibootmgr \
  lxqt lxqt-wayland-session \
  labwc kwin xorg-xwayland qt6-wayland \
  kscreen kscreenlocker plasma-desktop systemsettings powerdevil layer-shell-qt \
  gvfs mesa mesa-utils vulkan-intel vulkan-tools \
  networkmanager networkmanager-openrc network-manager-applet \
  blueman bluez bluez-openrc bluez-utils \
  alsa-utils pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber wireplumber-openrc pavucontrol-qt
```

---

## 9. GRUB (UEFI + legacy BIOS)

```bash
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix --removable
grub-install --target=i386-pc --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

---

## 10. Basic OpenRC services

```bash
rc-update add NetworkManager default
rc-update add bluetoothd default
```

(Do **not** enable fprintd here – that is covered in 03-biometrics.md)

---

## 11. Finish & reboot

```bash
exit
umount -R /mnt
reboot
```

After reboot, log in as **myname** + password, start the session with  startlxqtwayland or for X11 startx. Then continue with **02-post-install.md**.

---

*Artix OpenRC only – no systemd.*
