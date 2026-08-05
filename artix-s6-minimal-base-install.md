# Artix s6 Linux – Minimal Base Install → Plasma 6 + LXQt Panel

Artix base ISO (any recent)  
Target: **s6** init · TTY login · Minimal Plasma 6 (Wayland) + LXQt panel  
Hardware: Intel (adjust firmware/ucode if needed)

---

## 0. Check disk names
```bash
lsblk
```

## 1. Wipe & Partition
```bash
wipefs -a /dev/sda
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

## 2. Format Partitions
```bash
mkfs.fat -F32 /dev/sda1
mkfs.btrfs -f /dev/sda3
mkfs.btrfs -f /dev/sda4
```

## 3. Mount Partitions
```bash
mount /dev/sda3 /mnt
mkdir /mnt/home
mount /dev/sda4 /mnt/home
mkdir -p /mnt/boot
mount /dev/sda1 /mnt/boot
```

## 4. Network configuration (live ISO)
```bash
connmanctl
enable wifi
scan wifi
services
agent on
connect wifi_xxxxxxxxxx     # ← your network name from services
quit
```

Optional – time sync:
```bash
# depending on live ISO init
rc-service ntpd start   # or equivalent
```

Rank mirrors (optional but recommended):
```bash
pacman -Syy
pacman -S pacman-contrib
cp /etc/pacman.d/mirrorlist /etc/pacman.d/mirrorlist-artix
rankmirrors /etc/pacman.d/mirrorlist-artix > /etc/pacman.d/mirrorlist
```

## 5. Base system setup (bare minimum → TTY)
```bash
basestrap /mnt base base-devel s6-base elogind-s6 \
  linux linux-firmware nano
```

Generate fstab:
```bash
bash -c 'fstabgen -U /mnt > /mnt/etc/fstab'
```

## 6. Chroot – system settings
```bash
artix-chroot /mnt
```

Timezone:
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
echo "YOURHOSTNAME" > /etc/hostname
```

Hosts file:
```bash
nano /etc/hosts
```
```
127.0.0.1   localhost
::1         localhost
127.0.1.1   YOURHOSTNAME.localdomain YOURHOSTNAME
```

Root password:
```bash
passwd
```

Create user:
```bash
useradd -m -G wheel,video,audio,input,storage,optical yourusername
passwd yourusername
```

Sudo:
```bash
EDITOR=nano visudo
```
Uncomment:
```
%wheel ALL=(ALL:ALL) ALL
```

## 7. GRUB (UEFI + legacy BIOS)
```bash
pacman -S grub efibootmgr
grub-install --target=x86_64-efi --efi-directory=/boot --bootloader-id=Artix --removable
grub-install --target=i386-pc --recheck /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

## 8. Finish base install
```bash
exit
umount -R /mnt
reboot
```

---

# First boot – TTY login

Log in as your user (or root).

## 9. Install remaining packages

Update system first:
```bash
sudo pacman -Syu
```

### Full package list (adapted from original, OpenRC → s6)

```bash
sudo pacman -S \
  # Firmware & microcode (moved from basestrap)
  sof-firmware intel-ucode intel-media-driver libva-utils \
  \
  # Graphics
  mesa mesa-utils vulkan-intel vulkan-tools \
  \
  # Minimal Plasma 6 core (Wayland)
  plasma-desktop plasma-workspace kwin systemsettings \
  plasma-nm plasma-pa powerdevil bluedevil \
  kscreen kscreenlocker polkit-kde-agent \
  breeze breeze-icons \
  \
  # LXQt panel + required pieces
  lxqt-panel lxqt-globalkeys lxqt-notificationd lxqt-qtplugin lxqt-themes \
  libstatgrab libsysstat \
  \
  # Audio
  pipewire pipewire-alsa pipewire-pulse pipewire-jack wireplumber \
  pavucontrol-qt \
  \
  # Network
  networkmanager networkmanager-s6 network-manager-applet \
  \
  # Bluetooth
  bluez bluez-s6 bluez-utils blueman \
  \
  # Fingerprint (if you use it)
  fprintd \
  \
  # Minimal extras
  konsole dolphin gvfs \
  \
  # Optional but useful
  xorg-xwayland qt6-wayland
```

## 10. Enable s6 services

```bash
s6 set enable elogind
s6 set enable NetworkManager
s6 set enable bluetoothd
s6 set commit
s6 live install
```

(You can also enable `fprintd` if you installed it.)

## 11. Start the desktop (Wayland)

Create a simple start script:

```bash
nano ~/start-plasma
```

```bash
#!/bin/bash
export XDG_SESSION_TYPE=wayland
exec startplasma-wayland
```

Make executable:
```bash
chmod +x ~/start-plasma
```

Launch:
```bash
./start-plasma
```

### LXQt panel autostart

Once inside Plasma, create:

```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/lxqt-panel.desktop
```

```ini
[Desktop Entry]
Type=Application
Name=LXQt Panel
Exec=lxqt-panel
X-KDE-AutostartScript=true
```

Disable or remove Plasma’s default panel if you only want the LXQt one.

---

## Notes

- No display manager (SDDM) – pure TTY → startplasma-wayland
- All OpenRC packages replaced with s6 equivalents
- Firmware, media driver, mesa etc. moved out of basestrap for faster base install
- PipeWire/WirePlumber are started as user services / autostart (same as your original)
- Bluetooth, NetworkManager, elogind are supervised by s6

---

## Optional later additions

```bash
# If you want more Plasma tools
sudo pacman -S kdeplasma-addons kinfocenter

# If you prefer a different terminal / file manager later
# just install what you need
```

Done.  
You now have a clean, minimal Artix s6 system that boots to TTY and launches Plasma 6 + LXQt panel on demand.
