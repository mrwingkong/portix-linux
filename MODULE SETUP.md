# Modular .xzm Packages on Artix Linux & (Arch-based systems)

**Goal**: Turn Artix packages into self-contained, squashfs-based **.xzm** modules (inspired by Porteus / Slax style modularity).  

Install apps into read-only loop-mounted modules, keep the base system clean, easily add/remove apps, and make selected programs (like `mpv`) appear nicely in the menu with pseudo-GUI support.

**Target audience**: Users of minimal Artix/LXQt setups who want portable-ish / modular applications with hybrid containerization.

**Current status**: Proof-of-concept / personal workflow — 90% complete.

## Requirements

- Artix Linux or Arch-based distro with **pacman**
- `squashfs-tools` (provides `mksquashfs`)
- `sudo` privileges
- Working `pacman` & internet
- Desktop environment using **LXQt** (panel restart commands are LXQt-specific)
- OpenRC init system (for the boot auto-mount service)

Most commands will **fail gracefully** or show warnings if dependencies are missing.

## Quick Setup – One-time Preparation

```bash
# 1. Create needed directories
mkdir -p ~/modules ~/modules-mounts ~/.local/bin ~/.local/share/applications ~/bin

# 2. Add ~/bin to PATH (safe to run multiple times)
echo 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

_____________________________________________________________________________________________________
Core Tools
1. Module Builder – build-xzm.sh
This script downloads a package, extracts it, strips documentation/locales, and builds a compressed .xzm module.

# Create the script
cat > ~/bin/build-xzm.sh << 'EOF'
#!/bin/bash
set -euo pipefail

PKG="$1"
if [ -z "$PKG" ]; then
    echo "Usage: build-xzm.sh <package-name>"
    echo "Example: build-xzm.sh mpv"
    exit 1
fi

MODDIR="/tmp/mod-$PKG"
CACHEDIR="/tmp/pkgcache-$PKG"

rm -rf "$MODDIR" "$CACHEDIR"
mkdir -p "$MODDIR" "$CACHEDIR"

echo "Downloading $PKG..."
sudo pacman -Sw --noconfirm --cachedir="$CACHEDIR" "$PKG" || exit 1

echo "Extracting package..."
for file in "$CACHEDIR"/*.pkg.tar.zst; do
    [ -f "$file" ] || continue
    bsdtar -xf "$file" -C "$MODDIR" || true
done

echo "Stripping documentation & unnecessary files to keep module small..."
rm -rf "$MODDIR"/usr/share/{doc,man,info,locale,gtk-doc,help} 2>/dev/null || true
find "$MODDIR" -name '*.la' -delete 2>/dev/null || true

echo "Building SquashFS module (xz compression)..."
mksquashfs "$MODDIR" "$HOME/modules/00-$PKG.xzm" \
    -comp xz -Xdict-size 100% -b 1M -noappend

if [ -f "$HOME/modules/00-$PKG.xzm" ]; then
    echo "Module created successfully:"
    echo "→ $HOME/modules/00-$PKG.xzm"
    du -h "$HOME/modules/00-$PKG.xzm" | awk '{print "Size: " $1}'
else
    echo "ERROR: Module was not created."
    exit 1
fi

# Cleanup
rm -rf "$MODDIR" "$CACHEDIR"
sudo pacman -Scc --noconfirm >/dev/null 2>&1 || true

echo "Done."
EOF

chmod +x ~/bin/build-xzm.sh

_____________________________________________________________________________________________
2. Modular Package Manager Wrapper – pman
Installs/removes modules + creates menu entries + wrappers.

sudo tee /usr/local/bin/pman > /dev/null << 'EOF'
#!/bin/bash
set -euo pipefail

case "${1:-}" in
  install)
    shift
    if [ $# -eq 0 ]; then
      echo "Usage: pman install <package-name>"
      exit 1
    fi
    PKG="$1"
    echo "=== Building .xzm module for $PKG ==="
    build-xzm.sh "$PKG" || exit 1

    MOD_FILE="$HOME/modules/00-$PKG.xzm"
    [ -f "$MOD_FILE" ] || { echo "Module not found"; exit 1; }

    echo "Module size: $(du -h "$MOD_FILE" | cut -f1)"

    MNT_DIR="$HOME/modules-mounts/$PKG"
    mkdir -p "$MNT_DIR"

    echo "Mounting module read-only..."
    sudo mount -o loop,ro "$MOD_FILE" "$MNT_DIR" || exit 1
    sudo chown -R "$USER:$USER" "$MNT_DIR"

    # Try to extract useful .desktop metadata
    ORIG_DESKTOP=$(find "$MNT_DIR/usr/share/applications/" -name "*.desktop" -print -quit 2>/dev/null || true)
    if [ -n "$ORIG_DESKTOP" ] && [ -f "$ORIG_DESKTOP" ]; then
      ORIG_ICON=$(grep -m1 '^Icon=' "$ORIG_DESKTOP" | cut -d= -f2- || echo "utilities-terminal")
      ORIG_CATEGORIES=$(grep -m1 '^Categories=' "$ORIG_DESKTOP" | cut -d= -f2- || echo "Utility;")
      ORIG_NAME=$(grep -m1 '^Name=' "$ORIG_DESKTOP" | cut -d= -f2- || echo "$PKG")
    else
      ORIG_ICON="utilities-terminal"
      ORIG_CATEGORIES="Utility;"
      ORIG_NAME="$PKG"
    fi

    # Copy icons so they persist even if module is not mounted
    mkdir -p ~/.local/share/icons ~/.local/share/pixmaps
    cp -r "$MNT_DIR/usr/share/icons/"*   ~/.local/share/icons/   2>/dev/null || true
    cp -r "$MNT_DIR/usr/share/pixmaps/"* ~/.local/share/pixmaps/ 2>/dev/null || true

    # Create command wrapper
    WRAPPER="$HOME/.local/bin/$PKG"
    cat > "$WRAPPER" << EOW
#!/bin/bash
MOD="$MNT_DIR"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"

if [ "$PKG" = "mpv" ]; then
  if [ \$# -eq 0 ]; then
    # Menu launch → pseudo-GUI
    exec "\$MOD/usr/bin/mpv" --idle=yes --no-terminal --player-operation-mode=pseudo-gui --force-window=yes
  else
    exec "\$MOD/usr/bin/mpv" "\$@"
  fi
else
  exec "\$MOD/usr/bin/$PKG" "\$@"
fi
EOW
    chmod +x "$WRAPPER"

    # Create .desktop file
    DESKTOP="$HOME/.local/share/applications/$PKG.desktop"
    cat > "$DESKTOP" << EOD
[Desktop Entry]
Name=${ORIG_NAME} (modular)
Comment=Modular / read-only installation
Exec=$HOME/.local/bin/$PKG %F
Icon=${ORIG_ICON}
Terminal=false
Type=Application
Categories=${ORIG_CATEGORIES}
StartupNotify=true
EOD

    update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
    killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true

    echo "=== $PKG is now ready ==="
    echo "Run it with:   $PKG"
    echo "Or find it in the menu"
    ;;

  remove)
    shift
    if [ $# -eq 0 ]; then
      echo "Usage: pman remove <package-name>"
      exit 1
    fi
    PKG="$1"

    sudo umount "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rmdir "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rm -f "$HOME/modules/00-$PKG.xzm"
    rm -f "$HOME/.local/bin/$PKG"
    rm -f "$HOME/.local/share/applications/$PKG.desktop"

    # Optional: remove copied icons (careful – may delete shared icons)
    rm -rf ~/.local/share/icons/hicolor/*/apps/"$PKG".* 2>/dev/null || true
    rm -rf ~/.local/share/pixmaps/"$PKG".* 2>/dev/null || true

    update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
    killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true

    echo "Removed $PKG module and menu entries."
    ;;

  *)
    echo "pman – simple modular package manager for .xzm modules"
    echo
    echo "Usage:"
    echo "  pman install <package-name>     Build + mount + menu"
    echo "  pman remove <package-name>      Unmount + delete"
    echo
    ;;
esac
EOF

sudo chmod +x /usr/local/bin/pman

_______________________________________________________________________________________________________
3. MPV pseudo-GUI fix (optional – run after pman install mpv)

cat > ~/bin/customize-mpv-gui.sh << 'EOF'
#!/bin/bash
# Enhances mpv.desktop to always use pseudo-GUI when launched from menu

CONFIG_DIR="$HOME/.config/mpv"
CONFIG_FILE="$CONFIG_DIR/mpv.conf"
DESKTOP_FILE="$HOME/.local/share/applications/mpv.desktop"

mkdir -p "$CONFIG_DIR"

if ! grep -q "$$   pseudo-gui   $$" "$CONFIG_FILE" 2>/dev/null; then
  cat >> "$CONFIG_FILE" << 'EOC'
[pseudo-gui]
terminal=no
idle=yes
force-window=yes
player-operation-mode=pseudo-gui
EOC
  echo "Added [pseudo-gui] profile to mpv.conf"
fi

if grep -q "Exec=.*--profile=pseudo-gui" "$DESKTOP_FILE" 2>/dev/null; then
  echo "Desktop file already has --profile=pseudo-gui"
else
  sed -i '/^Exec=/ s/$/ --profile=pseudo-gui/' "$DESKTOP_FILE"
  echo "Added --profile=pseudo-gui to Exec line"
fi

update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true

echo "MPV menu entry updated for pseudo-GUI mode."
EOF

chmod +x ~/bin/customize-mpv-gui.sh

_____________________________________________________________________________________________________
4. Auto-mount all modules at boot (OpenRC)

sudo tee /etc/init.d/module-mounts > /dev/null << 'EOF'
#!/sbin/openrc-run

description="Auto-mount all 00-*.xzm modules at boot (silent)"

depend() {
    after localmount
    need bootmisc
}

start() {
    ebegin "Mounting .xzm modules"
    MODULES_DIR="/home/$USER/modules"      # ← change username if needed
    MOUNTS_DIR="/home/$USER/modules-mounts"

    mkdir -p "$MOUNTS_DIR" >/dev/null 2>&1

    for mod_file in "$MODULES_DIR"/00-*.xzm; do
        [ -f "$mod_file" ] || continue
        app_name=$(basename "$mod_file" .xzm | sed 's/^00-//')
        mnt_point="$MOUNTS_DIR/$app_name"

        mkdir -p "$mnt_point" >/dev/null 2>&1
        mountpoint -q "$mnt_point" && continue

        mount -o loop,ro "$mod_file" "$mnt_point" >/dev/null 2>&1 || continue
        chown -R "$USER:$USER" "$mnt_point" >/dev/null 2>&1 || true
    done
    eend 0
}

stop() {
    ebegin "Unmounting .xzm modules"
    for mnt in /home/$USER/modules-mounts/*; do
        [ -d "$mnt" ] || continue
        umount "$mnt" >/dev/null 2>&1
        rmdir "$mnt" >/dev/null 2>&1
    done
    eend 0
}
EOF

sudo chmod +x /etc/init.d/module-mounts
sudo rc-update add module-mounts default

Note: Replace $USER with your actual username (e.g. mrwingkong) or make it dynamic if you want to share the repo.
___________________________________________________________________________________________________

5. Fix missing icons after adding modules

mkdir -p ~/.config/autostart

cat > ~/.config/autostart/refresh-icons.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Refresh Icons & Panel
NoDisplay=true
Exec=/home/$USER/.config/autostart/refresh-icons.sh
EOF

cat > ~/.config/autostart/refresh-icons.sh << 'EOF'
#!/bin/bash
sleep 10
gtk-update-icon-cache ~/.local/share/icons/hicolor/  2>/dev/null || true
gtk-update-icon-cache /usr/share/icons/hicolor/     2>/dev/null || true
pkill lxqt-panel && lxqt-panel & disown
EOF

chmod +x ~/.config/autostart/refresh-icons.sh

(Again — replace $USER or hardcode your username.)
________________________________________________________________________________________________________

Usage Examples

pman install mpv
customize-mpv-gui.sh           # optional – improves menu behavior

pman install firefox
pman install vlc

pman remove vlc






# If something fails:
sudo rc-service module-mounts status
dmesg | grep -i mount

Done!
