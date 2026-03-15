# Modular .xzm Packages on Artix Linux & (Arch-based systems)

**Goal**: Turn Artix packages into self-contained, squashfs-based **.xzm** modules keeping base system updated (inspired by Porteus / Slax style modularity).  

Install apps into read-only loop-mounted modules, keep the base system clean, easily add/remove apps, and make programs appear nicely in the menu.

**Target audience**: Users of minimal Artix/LXQT setups who want portable-ish / modular applications.

**Current status**: Proof-of-concept / personal workflow — 90% complete. Hybrid module containerization.

## Requirements

- Artix Linux or Arch-based distro with **working internet & pacman**
- `--needed squashfs-tools libarchive gtk-update-icon-cache`
- `sudo` privileges
- Desktop environment using **LXQT** (panel restart commands are LXQt-specific)
- OpenRC init system (for the boot auto-mount service)

Most commands will **fail gracefully** or show warnings if dependencies are missing.

## Quick Setup – One-time Preparation

1. Create needed directories
```bash
mkdir -p ~/modules ~/modules-mounts ~/.local/bin ~/.local/share/applications ~/bin
````
2. Add ~/bin to PATH (safe to run multiple times)
```bash
echo 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc
````
## 1. Module Builder – build-xzm.sh
Module Builder Script — ~/bin/build-xzm.sh
```
nano ~/bin/build-xzm.sh
```
```
#!/bin/bash
PKG="$1"
if [ -z "$PKG" ]; then
  echo "Usage: build-xzm.sh <package>"
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
```
```
chmod +x ~/bin/build-xzm.sh
```
## 2. Basic wrapper for any app, extracts .desktop info if available, copies icons, etc.
```bash
sudo nano /usr/local/bin/pman
```
```
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
   
    # Find the original .desktop in the module (assume one main file)
    ORIG_DESKTOP=$(find "$MNT_DIR/usr/share/applications/" -name "*.desktop" -print -quit 2>/dev/null)
    if [ -f "$ORIG_DESKTOP" ]; then
      ORIG_ICON=$(grep '^Icon=' "$ORIG_DESKTOP" | cut -d= -f2 | head -1)
      ORIG_CATEGORIES=$(grep '^Categories=' "$ORIG_DESKTOP" | cut -d= -f2 | head -1)
      ORIG_NAME=$(grep '^Name=' "$ORIG_DESKTOP" | cut -d= -f2 | head -1)
    else
      ORIG_ICON="utilities-terminal"
      ORIG_CATEGORIES="Utility;"
      ORIG_NAME="$PKG"
    fi
   
    # Copy icons from module to local (for persistence without mount)
    mkdir -p "$HOME/.local/share/icons" "$HOME/.local/share/pixmaps"
    cp -r "$MNT_DIR/usr/share/icons/"* "$HOME/.local/share/icons/" 2>/dev/null
    cp -r "$MNT_DIR/usr/share/pixmaps/"* "$HOME/.local/share/pixmaps/" 2>/dev/null
   
    # Create generic wrapper
    WRAPPER="$HOME/.local/bin/$PKG"
    cat > "$WRAPPER" << EOF
#!/bin/bash
MOD="$MNT_DIR"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"
exec "\$MOD/usr/bin/$PKG" "\$@"
EOF
    chmod +x "$WRAPPER"
   
    # Create custom .desktop with extracted values
    DESKTOP="$HOME/.local/share/applications/$PKG.desktop"
    cat > "$DESKTOP" << EOF
[Desktop Entry]
Name=${ORIG_NAME:-$PKG} (modular)
Comment=Modular version
Exec=$HOME/.local/bin/$PKG %F
Icon=${ORIG_ICON:-utilities-terminal}
Terminal=false
Type=Application
Categories=${ORIG_CATEGORIES:-Utility;}
StartupNotify=true
EOF
   
    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    killall lxqt-panel 2>/dev/null
    lxqt-panel & >/dev/null 2>&1
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
   
    # Optional: Clean up copied icons
    rm -rf "$HOME/.local/share/icons/hicolor/*/apps/$PKG.*" 2>/dev/null
    rm -rf "$HOME/.local/share/pixmaps/$PKG.*" 2>/dev/null
   
    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    killall lxqt-panel 2>/dev/null
    lxqt-panel & >/dev/null 2>&1
    echo "Removed $PKG"
    ;;
  *)
    echo "Usage:"
    echo " pman install <app-name>"
    echo " pman remove <app-name>"
    ;;
esac
```
```
sudo chmod +x /usr/local/bin/pman
```
## 3. Auto-mount all modules at boot (OpenRC)
```bash
sudo nano /etc/init.d/module-mounts
```
```bash
#!/sbin/openrc-run
description="Auto-mount all .xzm modules at boot (silent)"
depend() {
    after localmount
    need bootmisc
}
start() {
    # No visible output at all
    MODULES_DIR="/home/mrwingkong/modules"
    MOUNTS_DIR="/home/mrwingkong/modules-mounts"
    mkdir -p "$MOUNTS_DIR" >/dev/null 2>&1
    for mod_file in "$MODULES_DIR"/00-*.xzm; do
        [ -f "$mod_file" ] || continue
        app_name=$(basename "$mod_file" .xzm | sed 's/^00-//')
        mnt_point="$MOUNTS_DIR/$app_name"
        mkdir -p "$mnt_point" >/dev/null 2>&1
        # Skip if already mounted
        mountpoint -q "$mnt_point" && continue
        # Mount silently
        mount -o loop,ro "$mod_file" "$mnt_point" >/dev/null 2>&1 || continue
        # Change ownership silently (necessary for user access)
        chown -R mrwingkong:mrwingkong "$mnt_point" >/dev/null 2>&1 || true
    done
    # No eend - silent success
    return 0
}
stop() {
    # Silent unmount
    for mnt in "$MOUNTS_DIR"/*; do
        [ -d "$mnt" ] || continue
        umount "$mnt" >/dev/null 2>&1
        rmdir "$mnt" >/dev/null 2>&1
    done
    return 0
}
```
Note: Replace $USER with your actual username (e.g. mrwingkong) or make it dynamic if you want to share the repo.
```bash
sudo chmod +x /etc/init.d/module-mounts
```
## add to default runlevel
```
sudo rc-update add module-mounts default
```
## 4. Icon cache refresh at login (fix missing icons after module add)
```bash
mkdir -p ~/.config/autostart
nano ~/.config/autostart/refresh-icons.sh
```
```bash
#!/bin/bash
sleep 10
gtk-update-icon-cache ~/.local/share/icons/hicolor/ 2>/dev/null || true
gtk-update-icon-cache /usr/share/icons/hicolor/ 2>/dev/null || true
pkill lxqt-panel && lxqt-panel & disown
```
```bash
chmod +x ~/.config/autostart/refresh-icons.sh
```
(Again — replace $USER or hardcode your username.)

___________________________________________________________________________________
___________________________________________________________________________________

## Usage Examples:
```
pman install mpv

Optional script - mpv-gui.sh - (loading of blank gui window)

pman remove mpv



## If something fails:

sudo rc-service module-mounts status

dmesg | grep -i mount

Done!

---------------------------------------------------------------------------------
## Some common fixes, hacks to improve specific applications:
MPV desktop shortcut to open a blank gui and stay open
```bash
nano ~/.config/mpv/mpv.conf
```
replace / add:
```bash
force-window     = yes
idle             = yes
terminal         = no
player-operation-mode = pseudo-gui     # ← this can help in some edge cases
keep-open        = yes                 # redundant with idle=yes but explicit
```

```bash
nano ~/.local/bin/gparted

#!/bin/bash
MOD="/home/mrwingkong/modules-mounts/gparted"  # ← fixed, no $HOME
export PATH="$MOD/bin:$MOD/sbin:$MOD/usr/bin:$MOD/usr/sbin:$PATH"
export LD_LIBRARY_PATH="$MOD/lib:$MOD/lib64:$MOD/usr/lib:$MOD/usr/lib64:$LD_LIBRARY_PATH"
exec "$MOD/usr/lib/gparted/gpartedbin" "$@"

chmod +x ~/.local/bin/gparted

sudo pacman -S lxqt-sudo


nano ~/.local/share/applications/gparted.desktop

Exec=lxqt-sudo /home/mrwingkong/.local/bin/gparted %F

Test:
lxqt-sudo /home/mrwingkong/.local/bin/gparted

