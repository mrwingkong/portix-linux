# Modular .xzm Packages on Arch Linux (or Arch-based systems)

Goal: Turn Arch packages into self-contained, squashfs-based .xzm modules (inspired by Porteus / Slax style modularity).  
Install apps into read-only loop-mounted modules, keep the base system clean, easily add/remove apps, and make selected programs (like mpv) appear nicely in the menu with pseudo-GUI support.

Target audience: Users of minimal Arch/LXQt setups who want portable-ish / modular applications without full containerization.

Current status: Proof-of-concept / personal workflow — not production-grade yet.

Requirements
• Arch Linux or Arch-based distro with pacman
• squashfs-tools          → sudo pacman -S squashfs-tools
• sudo privileges
• Working pacman & internet
• LXQt desktop (panel restart commands are LXQt-specific)
• OpenRC init system (for the auto-mount service)

Quick Setup – One-time Preparation

mkdir -p ~/modules ~/modules-mounts ~/.local/bin ~/.local/share/applications ~/bin

echo 'export PATH="$HOME/.local/bin:$HOME/bin:$PATH"' >> ~/.bashrc
source ~/.bashrc

1. Module Builder – ~/bin/build-xzm.sh

cat > ~/bin/build-xzm.sh << 'EOF'
#!/bin/bash
set -euo pipefail

PKG="$1"
[ -z "$PKG" ] && { echo "Usage: build-xzm.sh <package>"; exit 1; }

MODDIR="/tmp/mod-$PKG"
CACHEDIR="/tmp/pkgcache-$PKG"

rm -rf "$MODDIR" "$CACHEDIR"
mkdir -p "$MODDIR" "$CACHEDIR"

echo "Downloading $PKG..."
sudo pacman -Sw --noconfirm --cachedir="$CACHEDIR" "$PKG" || exit 1

echo "Extracting..."
for f in "$CACHEDIR"/*.pkg.tar.zst; do [ -f "$f" ] || continue; bsdtar -xf "$f" -C "$MODDIR" || true; done

echo "Stripping unnecessary files..."
rm -rf "$MODDIR"/usr/share/{doc,man,info,locale,gtk-doc,help} 2>/dev/null || true
find "$MODDIR" -name '*.la' -delete 2>/dev/null || true

echo "Creating module..."
mksquashfs "$MODDIR" "$HOME/modules/00-$PKG.xzm" -comp xz -Xdict-size 100% -b 1M -noappend

[ -f "$HOME/modules/00-$PKG.xzm" ] && {
    echo "Created: $HOME/modules/00-$PKG.xzm"
    du -h "$HOME/modules/00-$PKG.xzm" | awk '{print "Size: "$1}'
} || { echo "ERROR: module creation failed"; exit 1; }

rm -rf "$MODDIR" "$CACHEDIR"
sudo pacman -Scc --noconfirm >/dev/null 2>&1 || true
EOF

chmod +x ~/bin/build-xzm.sh

2. Main Wrapper – pman (install / remove)

sudo tee /usr/local/bin/pman >/dev/null << 'EOF'
#!/bin/bash
set -euo pipefail

case "${1:-}" in
  install)
    shift; [ $# -eq 0 ] && { echo "Usage: pman install <pkg>"; exit 1; }
    PKG="$1"
    build-xzm.sh "$PKG" || exit 1

    MOD_FILE="$HOME/modules/00-$PKG.xzm"
    [ -f "$MOD_FILE" ] || exit 1

    MNT_DIR="$HOME/modules-mounts/$PKG"
    mkdir -p "$MNT_DIR"
    sudo mount -o loop,ro "$MOD_FILE" "$MNT_DIR" || exit 1
    sudo chown -R "$USER:$USER" "$MNT_DIR"

    DESK=$(find "$MNT_DIR/usr/share/applications/" -name "*.desktop" -print -quit 2>/dev/null || true)
    if [ -n "$DESK" ] && [ -f "$DESK" ]; then
      ICON=$(grep -m1 '^Icon=' "$DESK" | cut -d= -f2- || echo "utilities-terminal")
      CATS=$(grep -m1 '^Categories=' "$DESK" | cut -d= -f2- || echo "Utility;")
      NAME=$(grep -m1 '^Name=' "$DESK" | cut -d= -f2- || echo "$PKG")
    else
      ICON="utilities-terminal"; CATS="Utility;"; NAME="$PKG"
    fi

    mkdir -p ~/.local/share/{icons,pixmaps}
    cp -r "$MNT_DIR"/usr/share/icons/*   ~/.local/share/icons/   2>/dev/null || true
    cp -r "$MNT_DIR"/usr/share/pixmaps/* ~/.local/share/pixmaps/ 2>/dev/null || true

    WRAPPER="$HOME/.local/bin/$PKG"
    cat > "$WRAPPER" << EOW
#!/bin/bash
MOD="$MNT_DIR"
export PATH="\$MOD/bin:\$MOD/sbin:\$MOD/usr/bin:\$MOD/usr/sbin:\$PATH"
export LD_LIBRARY_PATH="\$MOD/lib:\$MOD/lib64:\$MOD/usr/lib:\$MOD/usr/lib64:\$LD_LIBRARY_PATH"
if [ "$PKG" = "mpv" ] && [ \$# -eq 0 ]; then
  exec "\$MOD/usr/bin/mpv" --idle=yes --no-terminal --player-operation-mode=pseudo-gui --force-window=yes
else
  exec "\$MOD/usr/bin/$PKG" "\$@"
fi
EOW
    chmod +x "$WRAPPER"

    cat > "$HOME/.local/share/applications/$PKG.desktop" << EOD
[Desktop Entry]
Name=${NAME} (modular)
Comment=Modular installation
Exec=$HOME/.local/bin/$PKG %F
Icon=${ICON}
Terminal=false
Type=Application
Categories=${CATS}
StartupNotify=true
EOD

    update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
    killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true
    echo "$PKG installed. Run with: $PKG  or use menu."
    ;;

  remove)
    shift; [ $# -eq 0 ] && { echo "Usage: pman remove <pkg>"; exit 1; }
    PKG="$1"
    sudo umount "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rmdir "$HOME/modules-mounts/$PKG" 2>/dev/null || true
    rm -f "$HOME/modules/00-$PKG.xzm" "$HOME/.local/bin/$PKG" "$HOME/.local/share/applications/$PKG.desktop"
    rm -rf ~/.local/share/icons/hicolor/*/apps/"$PKG".* ~/.local/share/pixmaps/"$PKG".* 2>/dev/null || true
    update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
    killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true
    echo "Removed $PKG"
    ;;

  *) echo "Usage: pman install|remove <package>"; exit 1 ;;
esac
EOF

sudo chmod +x /usr/local/bin/pman

3. Optional MPV GUI fix (run after pman install mpv)

cat > ~/bin/customize-mpv-gui.sh << 'EOF'
#!/bin/bash
CONFIG_DIR="$HOME/.config/mpv"
mkdir -p "$CONFIG_DIR"
CONFIG="$CONFIG_DIR/mpv.conf"
DESK="$HOME/.local/share/applications/mpv.desktop"

grep -q "\[pseudo-gui\]" "$CONFIG" 2>/dev/null || cat >> "$CONFIG" << EOC
[pseudo-gui]
terminal=no
idle=yes
force-window=yes
player-operation-mode=pseudo-gui
EOC

sed -i '/^Exec=/ s/$/ --profile=pseudo-gui/' "$DESK" 2>/dev/null || true

update-desktop-database ~/.local/share/applications/ 2>/dev/null || true
killall lxqt-panel 2>/dev/null; lxqt-panel & >/dev/null 2>&1 || true
EOF

chmod +x ~/bin/customize-mpv-gui.sh

4. Auto-mount at boot (OpenRC)

sudo tee /etc/init.d/module-mounts >/dev/null << 'EOF'
#!/sbin/openrc-run
description="Mount .xzm modules at boot"

depend() { after localmount; need bootmisc; }

start() {
    ebegin "Mounting modules"
    for f in /home/$USER/modules/00-*.xzm; do
        [ -f "$f" ] || continue
        name=$(basename "$f" .xzm | sed 's/^00-//')
        mnt="/home/$USER/modules-mounts/$name"
        mkdir -p "$mnt"
        mountpoint -q "$mnt" && continue
        mount -o loop,ro "$f" "$mnt" >/dev/null 2>&1 && chown -R $USER:$USER "$mnt"
    done
    eend 0
}

stop() {
    ebegin "Unmounting modules"
    for m in /home/$USER/modules-mounts/*; do
        [ -d "$m" ] || continue
        umount "$m" >/dev/null 2>&1; rmdir "$m" 2>/dev/null
    done
    eend 0
}
EOF

sudo chmod +x /etc/init.d/module-mounts
sudo rc-update add module-mounts default

Note: Replace $USER with your actual username (e.g. mrwingkong) in the service file.

5. Icon refresh at login (fix missing icons)

mkdir -p ~/.config/autostart

cat > ~/.config/autostart/refresh-icons.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=Refresh Icons
NoDisplay=true
Exec=/home/$USER/.config/autostart/refresh-icons.sh
EOF

cat > ~/.config/autostart/refresh-icons.sh << 'EOF'
#!/bin/bash
sleep 10
gtk-update-icon-cache ~/.local/share/icons/hicolor/ 2>/dev/null || true
pkill lxqt-panel && lxqt-panel & disown
EOF

chmod +x ~/.config/autostart/refresh-icons.sh

Again: Replace $USER with your username.

Quick usage examples

pman install mpv && ~/bin/customize-mpv-gui.sh
pman install firefox
pman install vlc
pman remove vlc


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
