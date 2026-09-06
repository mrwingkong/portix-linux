# 10 — Breeze window theme (Portix)

Patched official **Breeze v6.7.4** for LXQt + KWin.

No Klassy. No Breeze Enhanced.

Copy **only the commands**. Do not copy ` ``` ` or the word `bash`.

## What you get

- Floating windows: tiny curve on all 4 corners
- Super+arrow / maximize / screen-edge snap: square corners
- 1px black outline (hides the gap, follows the curve)
- Small shadow on the active floating window only
- Title text hidden by colour scheme

## 1. Build packages

Already covered if you followed `01` and `02`. If CMake later says it cannot find ECM, run this:

```
sudo pacman -S --needed extra-cmake-modules cmake base-devel git \
  qt6-base qt6-tools kdecoration kf6-kcmutils kf6-ki18n \
  kf6-kguiaddons kf6-kwindowsystem kf6-kcoreaddons \
  kf6-kcolorscheme kf6-kconfig kf6-kconfigwidgets breeze
```

## 2. Get Breeze 6.7.4

Must be tag `v6.7.4`. Do not clone master.

```
cd ~
sudo rm -rf breeze
git clone --depth=1 --branch v6.7.4 https://invent.kde.org/plasma/breeze.git
cd ~/breeze
git describe --tags
```

Must print:

```
v6.7.4
```

## 3. Patch

Paste this whole block at a normal prompt.

```
python3 << 'PY'
from pathlib import Path
import re

h = Path.home() / "breeze/kdecoration/breeze.h"
cpp = Path.home() / "breeze/kdecoration/breezedecoration.cpp"
if not h.exists() or not cpp.exists():
    raise SystemExit("~/breeze is missing. Run step 2 first.")

ht = h.read_text()
ht, n = re.subn(
    r"static constexpr qreal Frame_FrameRadius = [^;]+;",
    "static constexpr qreal Frame_FrameRadius = 1.0;",
    ht,
    count=1,
)
if n != 1:
    raise SystemExit("Could not set Frame_FrameRadius")
h.write_text(ht)
print("OK: tiny radius")

t = cpp.read_text()

old_radius = """    qreal topLeftRightRadius = 0;
    qreal bottomLeftRadius = 0;
    qreal bottomRightRadius = 0;
    if (m_internalSettings->roundedCorners()) {
        if (hideTitleBar()) {
            topLeftRightRadius = m_scaledCornerRadius;
        }

        if (hasNoBorders()) {
            if (!isBottomEdge()) {
                if (!isLeftEdge()) {
                    bottomLeftRadius = m_scaledCornerRadius;
                }
                if (!isRightEdge()) {
                    bottomRightRadius = m_scaledCornerRadius;
                }
            }
        }
    }
    setBorderRadius(KDecoration3::BorderRadius(topLeftRightRadius, topLeftRightRadius, bottomRightRadius, bottomLeftRadius));"""

new_radius = """    qreal topLeftRadius = 0;
    qreal topRightRadius = 0;
    qreal bottomLeftRadius = 0;
    qreal bottomRightRadius = 0;

    const bool snapped = isMaximized()
        || isMaximizedHorizontally()
        || isMaximizedVertically()
        || isLeftEdge()
        || isRightEdge()
        || isTopEdge()
        || isBottomEdge()
        || window()->isMaximized()
        || window()->isMaximizedHorizontally()
        || window()->isMaximizedVertically()
        || window()->adjacentScreenEdges();

    if (!snapped) {
        topLeftRadius = m_scaledCornerRadius;
        topRightRadius = m_scaledCornerRadius;
        bottomLeftRadius = m_scaledCornerRadius;
        bottomRightRadius = m_scaledCornerRadius;
    }

    setBorderRadius(KDecoration3::BorderRadius(topLeftRadius, topRightRadius, bottomRightRadius, bottomLeftRadius));"""

if old_radius not in t:
    raise SystemExit("Could not find corner-radius block")
t = t.replace(old_radius, new_radius, 1)
print("OK: floating vs snapped clip radius")

old_frame = """        painter->setBrush(window()->color(window()->isActive() ? ColorGroup::Active : ColorGroup::Inactive, ColorRole::Frame));"""
new_frame = """        painter->setBrush(QColor(0, 0, 0));"""
if old_frame not in t:
    raise SystemExit("Could not find Frame brush")
t = t.replace(old_frame, new_frame, 1)
print("OK: frame paint black")

old_tb = """    if (isMaximized() || !settings()->isAlphaChannelSupported()) {"""
new_tb = """    if (isMaximized() || isMaximizedHorizontally() || isMaximizedVertically()
        || isLeftEdge() || isRightEdge() || isTopEdge() || isBottomEdge()
        || window()->adjacentScreenEdges()
        || !settings()->isAlphaChannelSupported()) {"""
if old_tb not in t:
    raise SystemExit("Could not find paintTitleBar check")
t = t.replace(old_tb, new_tb, 1)
print("OK: tiled titlebars paint square")

old_col = """        const auto color = KColorUtils::mix(window()->color(window()->isActive() ? ColorGroup::Active : ColorGroup::Inactive, ColorRole::Frame),
                                            window()->palette().text().color(),
                                            KColorScheme::frameContrast());"""
new_col = """        const auto color = QColor(0, 0, 0);"""
if old_col not in t:
    raise SystemExit("Could not find outline colour")
t = t.replace(old_col, new_col, 1)
print("OK: outline black")

old_ol = """        qreal topLeftRightRadius = 0;
        qreal bottomLeftRadius = 0;
        qreal bottomRightRadius = 0;
        if (!hideTitleBar() || m_internalSettings->roundedCorners()) {
            topLeftRightRadius = m_scaledCornerRadius;
        }
        if (!hasNoBorders() || m_internalSettings->roundedCorners()) {
            bottomLeftRadius = m_scaledCornerRadius;
            bottomRightRadius = m_scaledCornerRadius;
        }

        const auto radius = KDecoration3::BorderRadius(topLeftRightRadius, topLeftRightRadius, bottomRightRadius, bottomLeftRadius);
        setBorderOutline(KDecoration3::BorderOutline(thickness, color, radius));"""

new_ol = """        qreal olTopLeft = 0;
        qreal olTopRight = 0;
        qreal olBottomLeft = 0;
        qreal olBottomRight = 0;
        const bool snappedOl = isMaximized()
            || isMaximizedHorizontally()
            || isMaximizedVertically()
            || isLeftEdge()
            || isRightEdge()
            || isTopEdge()
            || isBottomEdge()
            || window()->adjacentScreenEdges();
        if (!snappedOl) {
            olTopLeft = m_scaledCornerRadius;
            olTopRight = m_scaledCornerRadius;
            olBottomLeft = m_scaledCornerRadius;
            olBottomRight = m_scaledCornerRadius;
        }
        const auto radius = KDecoration3::BorderRadius(olTopLeft, olTopRight, olBottomRight, olBottomLeft);
        setBorderOutline(KDecoration3::BorderOutline(thickness, color, radius));"""

if old_ol not in t:
    raise SystemExit("Could not find outline radius block")
t = t.replace(old_ol, new_ol, 1)
print("OK: outline follows corners")

old_shadow = """    auto &shadow = (window()->isActive()) ? g_sShadow : g_sShadowInactive;
    if (!shadow) {
        g_sShadow = createShadowObject(1.0);
        g_sShadowInactive = createShadowObject(0.5);
    }
    setShadow(shadow);
}"""

new_shadow = """    const bool snappedOrMaximized =
        isMaximized() || isLeftEdge() || isRightEdge() || isTopEdge() || isBottomEdge()
        || window()->adjacentScreenEdges();

    if (!window()->isActive() || snappedOrMaximized) {
        setShadow(nullptr);
        return;
    }

    if (!g_sShadow) {
        g_sShadow = createShadowObject(1.0);
    }
    setShadow(g_sShadow);
}"""

if old_shadow not in t:
    raise SystemExit("Could not find updateShadow ending")
t = t.replace(old_shadow, new_shadow, 1)
print("OK: shadow only on active floating window")

cpp.write_text(t)
print("ALL EDITS DONE")
PY
```

Must print `ALL EDITS DONE`.

## 4. Build

```
cd ~/breeze
rm -rf build
mkdir build
cd build
cmake .. -DCMAKE_INSTALL_PREFIX=/usr -DCMAKE_BUILD_TYPE=Release -DBUILD_TESTING=OFF -DBUILD_QT6=ON -DBUILD_QT5=OFF
make -j$(nproc)
sudo make install
ls -l /usr/lib64/qt6/plugins/org.kde.kdecoration3/org.kde.breeze.so
```

The `.so` date must be today.

```
kwin_wayland --replace &
```

Or log out and back in.

## 5. Settings after it builds

Window Decorations

- Theme: Breeze
- Border size: No window borders

Breeze configure → General

- Draw border on maximised and tiled windows: off
- Draw titlebar background gradient: off
- Round bottom corners of windows with no borders: off

Breeze configure → Shadows and Outline

- Shadow size: Small
- Draw window outline: on

Colours

- Breeze Classic 2 (only after you copy the file below)

## 6. Colour scheme (your file, not shipped)

Breeze Classic 2 is not on a fresh install. Copy it from the old machine or from this repo:

```
mkdir -p ~/.local/share/color-schemes
cp BreezeClassic2.colors ~/.local/share/color-schemes/
```

Then set `~/.config/kdeglobals`:

```
[General]
ColorScheme=BreezeClassic2

[WM]
activeBackground=0,0,0
activeBlend=0,0,0
activeForeground=0,0,0
inactiveBackground=0,0,0
inactiveBlend=0,0,0
inactiveForeground=0,0,0
```

Optional tile gap:

```
kwriteconfig6 --file kwinrc --group Tiling --key padding 0
```

## After a breeze package update

Run from step 2 again. Always use tag `v6.7.4` until Portix moves Plasma version.

Check versions with:

```
pacman -Q breeze kdecoration kwin
```
