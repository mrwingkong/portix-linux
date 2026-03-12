# Modular .xzm Packages on Arch Linux (or Arch-based systems)

**Goal**: Turn Arch packages into self-contained, squashfs-based **.xzm** modules (inspired by Porteus / Slax style modularity).  

Install apps into read-only loop-mounted modules, keep the base system clean, easily add/remove apps, and make selected programs (like `mpv`) appear nicely in the menu with pseudo-GUI support.

**Target audience**: Users of minimal Arch/LXQt setups who want portable-ish / modular applications without full containerization.

**Current status**: Proof-of-concept / personal workflow — not production-grade yet.

## Requirements

- Arch Linux or Arch-based distro with **pacman**
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

# Core Tools
## 1. Module Builder – build-xzm.sh
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
