#!/bin/bash
# ================================================
# PORTIX PMAN FULL SETUP + AUTO-MOUNT PERSISTENCE
# ================================================
# Single copy-paste script.
# Works for any user.
# pman uses coloured output (yellow for progress).
# build-xzm.sh now also uses yellow for all progress steps
# (including the parts around the mksquashfs "Creating 4.0 filesystem" bar).

set -e

# Detect current user for generic setup
USER=$(whoami)
HOMEDIR=$(eval echo ~$USER)

echo "=== Portix pman FULL SETUP + BOOT PERSISTENCE ==="

# 1. Directories + PATH
mkdir -p "$HOMEDIR/modules" "$HOMEDIR/modules-mounts" "$HOMEDIR/.local/bin" \
         "$HOMEDIR/.local/share/applications" "$HOMEDIR/bin" "$HOMEDIR/.config/autostart"

if ! grep -q 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' "$HOMEDIR/.bashrc"; then
    echo 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' >> "$HOMEDIR/.bashrc"
fi
source "$HOMEDIR/.bashrc"

# 2. build-xzm.sh WITH YELLOW PROGRESS (around the mksquashfs bar)
cat > "$HOMEDIR/bin/build-xzm.sh" << 'EOF'
#!/bin/bash
YELLOW='\033[1;33m'
GREEN='\033[1;32m'
RED='\033[1;31m'
NC='\033[0m'

PKG="$1"
if [ -z "$PKG" ]; then echo -e "${RED}Usage: build-xzm.sh <package>${NC}"; exit 1; fi

MODDIR="/tmp/mod-$PKG"
CACHEDIR="/tmp/pkgcache-$PKG"
rm -rf "$MODDIR" "$CACHEDIR"
mkdir -p "$MODDIR" "$CACHEDIR"

echo -e "${YELLOW}Downloading $PKG...${NC}"
sudo pacman -Sw --noconfirm --cachedir="$CACHEDIR" "$PKG" || exit 1

echo -e "${YELLOW}Extracting...${NC}"
for file in "$CACHEDIR"/*.pkg.tar.zst; do [ -f "$file" ] || continue; bsdtar -xf "$file" -C "$MODDIR" || true; done

echo -e "${YELLOW}Stripping...${NC}"
rm -rf "$MODDIR"/usr/share/{doc,man,info,locale,gtk-doc,help} 2>/dev/null
find "$MODDIR" -name '*.la' -delete 2>/dev/null

echo -e "${YELLOW}Creating module...${NC}"
mksquashfs "$MODDIR" "$HOME/modules/00-$PKG.xzm" -comp xz -Xdict-size 100% -b 1M -noappend

if [ -f "$HOME/modules/00-$PKG.xzm" ]; then
  echo -e "${GREEN}Module created: $HOME/modules/00-$PKG.xzm${NC}"
  echo -e "${GREEN}Size: $(du -h "$HOME/modules/00-$PKG.xzm" | cut -f1)${NC}"
else
  echo -e "${RED}ERROR: Module not created${NC}"
  exit 1
fi

rm -rf "$MODDIR" "$CACHEDIR"
sudo pacman -Scc --noconfirm >/dev/null 2>&1
EOF
chmod +x "$HOMEDIR/bin/build-xzm.sh"

# 3. pman with coloured progress output (yellow for all progress steps)
cat > /tmp/pman-fixed << 'EOF'
#!/bin/bash
YELLOW='\033[1;33m'
GREEN='\033[1;32m'
RED='\033[1;31m'
NC='\033[0m'  # No Colour

if [ "$(whoami)" = "root" ]; then
  echo -e "${RED}ERROR: pman must be run as your normal user, NOT root!${NC}"
  exit 1
fi

case "$1" in
  install)
    shift
    [ $# -eq 0 ] && { echo -e "${RED}Usage: pman install <app-name>${NC}"; exit 1; }
    PKG="$1"

    echo -e "${YELLOW}=== Building module for $PKG ===${NC}"
    build-xzm.sh "$PKG" || exit 1

    MOD_FILE="$HOME/modules/00-$PKG.xzm"
    [ -f "$MOD_FILE" ] || { echo -e "${RED}Module not created${NC}"; exit 1; }

    MNT_DIR="$HOME/modules-mounts/$PKG"
    mkdir -p "$MNT_DIR"
    echo -e "${YELLOW}Mounting...${NC}"
    sudo mount -o loop,ro "$MOD_FILE" "$MNT_DIR" || exit 1
    sudo chown -R "$USER:$USER" "$MNT_DIR"

    # Grab original .desktop info
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

    # Copy icons
    mkdir -p "$HOME/.local/share/icons" "$HOME/.local/share/pixmaps"
    cp -r "$MNT_DIR/usr/share/icons/"* "$HOME/.local/share/icons/" 2>/dev/null || true
    cp -r "$MNT_DIR/usr/share/pixmaps/"* "$HOME/.local/share/pixmaps/" 2>/dev/null || true

    # === WRAPPER ===
    WRAPPER="$HOME/.local/bin/$PKG"
    if [ "$PKG" = "gparted" ]; then
      cat > "$WRAPPER" << WRAPPERSCRIPT
#!/bin/bash
MOD="$HOME/modules-mounts/gparted"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"
exec "\$MOD/usr/lib/gparted/gpartedbin" "\$@"
WRAPPERSCRIPT
    else
      cat > "$WRAPPER" << WRAPPERSCRIPT
#!/bin/bash
MOD="$HOME/modules-mounts/$PKG"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"
exec "\$MOD/usr/bin/$PKG" "\$@"
WRAPPERSCRIPT
    fi
    chmod +x "$WRAPPER"

    # === .desktop ===
    DESKTOP="$HOME/.local/share/applications/$PKG.desktop"
    if [ "$PKG" = "gparted" ]; then
      cat > "$DESKTOP" << DESKTOP_EOF
[Desktop Entry]
Name=${ORIG_NAME:-$PKG} (modular)
Comment=Modular version
Exec=lxqt-sudo $HOME/.local/bin/$PKG %F
Icon=${ORIG_ICON:-utilities-terminal}
Terminal=false
Type=Application
Categories=${ORIG_CATEGORIES:-Utility;}
StartupNotify=true
DESKTOP_EOF
    else
      cat > "$DESKTOP" << DESKTOP_EOF
[Desktop Entry]
Name=${ORIG_NAME:-$PKG} (modular)
Comment=Modular version
Exec=$HOME/.local/bin/$PKG %F
Icon=${ORIG_ICON:-utilities-terminal}
Terminal=false
Type=Application
Categories=${ORIG_CATEGORIES:-Utility;}
StartupNotify=true
DESKTOP_EOF
    fi
    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    killall lxqt-panel 2>/dev/null || true
    lxqt-panel & >/dev/null 2>&1

    echo -e "${GREEN}=== $PKG installed and ready ===${NC}"
    ;;
  remove)
    shift
    PKG="$1"
    echo -e "${YELLOW}Removing $PKG...${NC}"
    sudo umount "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rmdir "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rm -f "$HOME/modules/00-$PKG.xzm"
    rm -f "$HOME/.local/bin/$PKG"
    rm -f "$HOME/.local/share/applications/$PKG.desktop"
    rm -rf "$HOME/.local/share/icons/hicolor"/*/apps/"$PKG."* 2>/dev/null || true
    rm -rf "$HOME/.local/share/pixmaps/$PKG."* 2>/dev/null || true
    update-desktop-database "$HOME/.local/share/applications/" 2>/dev/null
    killall lxqt-panel 2>/dev/null || true
    lxqt-panel & >/dev/null 2>&1
    echo -e "${GREEN}$PKG removed${NC}"
    ;;
  *)
    echo -e "${RED}Usage: pman install <app> or pman remove <app>${NC}"
    ;;
esac
EOF
sudo mv /tmp/pman-fixed /usr/local/bin/pman
sudo chmod +x /usr/local/bin/pman
echo "pman installed to /usr/local/bin/pman"

# 4. OpenRC service (generic for any user)
echo "Setting up OpenRC module-mounts service..."
cat > /tmp/module-mounts << EOF
#!/sbin/openrc-run
description="Auto-mount all .xzm modules at boot (silent)"
depend() {
    after localmount
    need bootmisc
}
start() {
    MODULES_DIR="$HOMEDIR/modules"
    MOUNTS_DIR="$HOMEDIR/modules-mounts"
    mkdir -p "\$MOUNTS_DIR" >/dev/null 2>&1
    for mod_file in "\$MODULES_DIR"/00-*.xzm; do
        [ -f "\$mod_file" ] || continue
        app_name=\$(basename "\$mod_file" .xzm | sed 's/^00-//')
        mnt_point="\$MOUNTS_DIR/\$app_name"
        mkdir -p "\$mnt_point" >/dev/null 2>&1
        mountpoint -q "\$mnt_point" && continue
        mount -o loop,ro "\$mod_file" "\$mnt_point" >/dev/null 2>&1 || continue
        chown -R $USER:$USER "\$mnt_point" >/dev/null 2>&1 || true
    done
    return 0
}
stop() {
    MOUNTS_DIR="$HOMEDIR/modules-mounts"
    for mnt in "\$MOUNTS_DIR"/*; do
        [ -d "\$mnt" ] || continue
        umount "\$mnt" >/dev/null 2>&1
        rmdir "\$mnt" >/dev/null 2>&1
    done
    return 0
}
EOF
sudo mv /tmp/module-mounts /etc/init.d/module-mounts
sudo chmod +x /etc/init.d/module-mounts
sudo rc-update add module-mounts default
echo "module-mounts service added to default runlevel"

# 5. LXQt autostart icon refresh
cat > "$HOMEDIR/.config/autostart/refresh-icons.sh" << 'EOF'
#!/bin/bash
sleep 10
gtk-update-icon-cache "$HOME/.local/share/icons/hicolor/" 2>/dev/null || true
gtk-update-icon-cache /usr/share/icons/hicolor/ 2>/dev/null || true
pkill lxqt-panel && lxqt-panel & disown
EOF
chmod +x "$HOMEDIR/.config/autostart/refresh-icons.sh"

echo "=== SETUP COMPLETE ==="
echo ""
echo "NEXT STEPS:"
echo "1. Reboot (or run: sudo rc-service module-mounts restart)"
echo "2. Rebuild your apps:"
echo "   pman remove <app> && pman install <app>"
echo ""
echo "Yellow progress messages now appear during module creation"
echo "(including around the mksquashfs \"Creating 4.0 filesystem\" bar)."
