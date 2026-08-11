# Plasma Keyboard (Virtual Keyboard) for Tablet Mode – LXQt + KWin Wayland

Enable the official **Plasma Keyboard** so it appears when you tap text fields in tablet mode.

---

## 1. Install

```bash
sudo pacman -S plasma-keyboard
```

If the package name differs on your mirror:

```bash
pacman -Ss plasma-keyboard
# or
pacman -Ss keyboard | grep -i plasma
```

---

## 2. Point KWin at Plasma Keyboard

```bash
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod "/usr/share/applications/org.kde.plasma.keyboard.desktop"
```

Verify the desktop file exists:

```bash
ls /usr/share/applications/*plasma*keyboard*
# Expected: org.kde.plasma.keyboard.desktop
```

If the path is different, use the one that exists.

---

## 3. Optional – always show the keyboard

By default KWin only shows the virtual keyboard on **touch** interaction with a text field.

To force it more aggressively you can set:

```bash
# Temporary test for current session
export KWIN_IM_SHOW_ALWAYS=1
```

To make it permanent, add to environment:

```bash
mkdir -p ~/.config/environment.d
echo 'KWIN_IM_SHOW_ALWAYS=1' >> ~/.config/environment.d/plasma-keyboard.conf
```

(Only use this if the keyboard refuses to appear on touch.)

---

## 4. Apply the change

Log out and log back in (or reboot).

---

## 5. Test

1. Fold the device into tablet mode (or just use the touchscreen).
2. Tap any text field (browser address bar, terminal, text editor, etc.).
3. The Plasma Keyboard should slide up.

---

## 6. Disable / switch away

```bash
# Turn virtual keyboard off
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod ""

# Or point back to nothing / another keyboard
```

Then log out/in.

---

## 7. Troubleshooting

| Problem | Fix |
|---------|-----|
| Keyboard never appears | Confirm `InputMethod` path is correct and log out/in |
| Only appears with mouse click, not touch | Touch events may not be reaching the field; try `KWIN_IM_SHOW_ALWAYS=1` |
| Wrong desktop file path | `ls /usr/share/applications/*keyboard*` and update the `kwriteconfig6` line |
| Conflicts with Maliit | Make sure Maliit is removed or its InputMethod entry is cleared |

Check current setting:

```bash
kreadconfig6 --file kwinrc --group Wayland --key InputMethod
```

---

## 8. Notes for LXQt + KWin

- Plasma Keyboard is the current recommended virtual keyboard for KWin Wayland (Maliit is largely unmaintained on Arch/Artix).
- It appears only when an input method is requested (normally on touch focus of a text field).
- It does **not** automatically add media/F-keys; it is a typing keyboard.
- Brightness/volume in tablet mode still rely on your existing sxhkd + OSD scripts or panel buttons.

---

*Matches the Plasma Keyboard setup used with KWin Wayland on Artix/Arch + LXQt.*
