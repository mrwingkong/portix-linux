# Artix OpenRC + LXQt — Modular .xzm Packages Setup

Guide to create, manage and auto-mount portable squashfs modules (.xzm) for selected applications.  
Keeps /usr clean, allows easy removal, fast testing of software.

**Features**
- Build .xzm from single pacman package
- Auto-create wrapper + .desktop entry
- Mount at boot via OpenRC service
- pman helper command (install / remove)
- Works nicely with LXQt

Run these steps after base + LXQt install is complete.

## 1. Create Directories

mkdir -p ~/modules ~/modules-mounts ~/.local/bin ~/.local/share/applications ~/bin

## 2. Add ~/bin to PATH (safe to re-run)

echo 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

## 3. Module Builder Script — ~/bin/build-xzm.sh

nano ~/bin/build-xzm.sh

# Paste the following content:

#!/bin/bash

PKG="$1"
if [ -z "$PKG" ]; then
  echo "Usage: build-xzm.sh <app-name>"
  exit 1
fi

MODDIR="/tmp/mod-$PKG"
CACHEDIR="/tmp/pkgcache-$PKG"

rm -rf "$MODDIR" "$CACHEDIR"
mkdir -p "$MODDIR" "$CACHEDIR"

echo "Downloading $PKG..."
sudo pacman -Sw --noconfirm --cachedir="$CACHEDIR" "$PKG" || exit 1

echo "Extracting..."
for file in "$CACHEDIR"/*.pkg.tar.zst; do
  [ -f "$file" ] || continue
  bsdtar -xf "$file" -C "$MODDIR" || true
done

echo "Stripping to keep small..."
rm -rf "$MODDIR"/usr/share/{doc,man,info,locale,gtk-doc,help} 2>/dev/null
find "$MODDIR" -name '*.la' -delete 2>/dev/null

echo "Creating module..."
mksquashfs "$MODDIR" "$HOME/modules/00-$PKG.xzm" -comp xz -Xdict-size 100% -b 1M -noappend

if [ -f "$HOME/modules/00-$PKG.xzm" ]; then
  echo "Module created: $HOME/modules/00-$PKG.xzm"
  echo "Size: $(du -h "$HOME/modules/00-$PKG.xzm" | cut -f1)"
else
  echo "ERROR: Module file not created"
  exit 1
fi

rm -rf "$MODDIR" "$CACHEDIR"
sudo pacman -Scc --noconfirm >/dev/null 2>&1

# Make executable
chmod +x ~/bin/build-xzm.sh

## 4. pman — modular package manager wrapper (/usr/local/bin/pman)

sudo nano /usr/local/bin/pman

# Paste the following:

#!/bin/bash

case "$1" in
  install)
    shift
    if [ $# -eq 0 ]; then
      echo "Usage: pman install <app-name>"
      exit 1
    fi

    PKG="$1"

    echo "=== Building module for $PKG ==="

    build-xzm.sh "$PKG" || exit 1

    MOD_FILE="$HOME/modules/00-$PKG.xzm"
    if [ ! -f "$MOD_FILE" ]; then
      echo "Module not created"
      exit 1
    fi

    echo "Size: $(du -h "$MOD_FILE" | cut -f1)"

    MNT_DIR="$HOME/modules-mounts/$PKG"
    mkdir -p "$MNT_DIR"

    echo "Mounting module..."
    sudo mount -o loop,ro "$MOD_FILE" "$MNT_DIR" || exit 1

    sudo chown -R "$USER:$USER" "$MNT_DIR"

    WRAPPER="$HOME/.local/bin/$PKG"
    cat > "$WRAPPER" << EOF
#!/bin/bash
MOD="$MNT_DIR"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"

if [ "$PKG" = "mpv" ]; then
  exec "\$MOD/usr/bin/mpv" --idle=yes --no-terminal --player-operation-mode=pseudo-gui --force-window=yes "\$@"
else
  exec "\$MOD/usr/bin/$PKG" "\$@"
fi
EOF

    chmod +x "$WRAPPER"

    DESKTOP="$HOME/.local/share/applications/$PKG.desktop"
    cat > "$DESKTOP" << EOF
[Desktop Entry]
Name=$PKG (modular)
Comment=Modular version
Exec=$HOME/.local/bin/$PKG %F
Icon=utilities-terminal
Terminal=false
Type=Application
Categories=Utility;
StartupNotify=true
EOF

    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    lxqt-panel --restart 2>/dev/null
    echo "=== $PKG ready ==="
    echo "Run: $PKG"
    echo "Or check menu"
    ;;

  remove)
    shift
    PKG="$1"

    sudo umount "$HOME/modules-mounts/$PKG" 2>/dev/null
    rmdir "$HOME/modules-mounts/$PKG" 2>/dev/null
    rm -f "$HOME/modules/00-$PKG.xzm"
    rm -f "$HOME/.local/bin/$PKG"
    rm -f "$HOME/.local/share/applications/$PKG.desktop"
    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    lxqt-panel --restart 2>/dev/null
    echo "Removed $PKG"
    ;;

  *)
    echo "Usage:"
    echo "  pman install <app-name>"
    echo "  pman remove <app-name>"
    ;;
esac

# Make executable
sudo chmod +x /usr/local/bin/pman

## 5. Auto-mount all modules at boot (OpenRC service)

sudo nano /etc/init.d/module-mounts

# Paste:

#!/sbin/openrc-run

description="Auto-mount all .xzm modules at boot"

depend() {
  after localmount
  need bootmisc
}

start() {
  ebegin "Mounting modules"

  MODULES_DIR="/home/mrwingkong/modules"
  MOUNTS_DIR="/home/mrwingkong/modules-mounts"

  mkdir -p "$MOUNTS_DIR"

  for mod_file in "$MODULES_DIR"/00-*.xzm; do
    [ -f "$mod_file" ] || continue

    app_name=$(basename "$mod_file" .xzm | sed 's/^00-//')

    mnt_point="$MOUNTS_DIR/$app_name"

    mkdir -p "$mnt_point"

    if mount | grep -q "$mnt_point"; then
      continue
    fi

    mount -o loop,ro "$mod_file" "$mnt_point" || continue
    chown -R mrwingkong:mrwingkong "$mnt_point" 2>/dev/null
  done

  eend $?
}

stop() {
  ebegin "Unmounting modules"
  for mnt in "$MOUNTS_DIR"/*; do
    [ -d "$mnt" ] || continue
    umount "$mnt" 2>/dev/null
    rmdir "$mnt" 2>/dev/null
  done
  eend $?
}

# Make executable & enable at boot
sudo chmod +x /etc/init.d/module-mounts
sudo rc-update add module-mounts default

# Test manually
sudo rc-service module-mounts start

# Check mounts
mount | grep modules-mounts

## 6. Icon cache refresh at login (fix missing icons after module add)

nano ~/.config/autostart/refresh-icons.sh

# Paste:

#!/bin/bash

sleep 10

gtk-update-icon-cache ~/.local/share/icons/hicolor/ 2>/dev/null || true
gtk-update-icon-cache /usr/share/icons/hicolor/ 2>/dev/null || true
pkill lxqt-panel && lxqt-panel & disown

chmod +x ~/.config/autostart/refresh-icons.sh

nano ~/.config/autostart/refresh-icons.desktop

# Paste (replace /home/USERNAME with your actual $HOME):

[Desktop Entry]
Type=Application
Exec=/home/USERNAME/.config/autostart/refresh-icons.sh
Hidden=false
NoDisplay=false
X-GNOME-Autostart-enabled=true
Name=Refresh Icons
Comment=Update icon cache and restart panel

## 7. Quick Test

# Example usage
pman install mousepad
pman install mpv

# Reboot
reboot

# After login — check:
mount | grep modules-mounts

# Try launching
mousepad
mpv some-video.mp4     # should open GUI window

# If something fails:
sudo rc-service module-mounts status
dmesg | grep -i mount

Done!
