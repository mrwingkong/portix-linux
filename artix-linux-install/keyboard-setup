# Set UK Keyboard Layout as Default – LXQt + KWin Wayland

Under LXQt + KWin Wayland the normal “Keyboard” settings page often cannot set a persistent UK layout. Use the methods below instead.

UK layout code = **`gb`**

---

## 1. System-wide (recommended)

```bash
sudo localectl set-x11-keymap gb
sudo localectl set-keymap uk
```

Check:

```bash
localectl status
```

You should see something like:

```
X11 Layout: gb
VC Keymap: uk
```

---

## 2. KWin / Plasma keyboard config

```bash
# Create or edit the KDE keyboard config
mkdir -p ~/.config

cat > ~/.config/kxkbrc << 'EOF'
[Layout]
DisplayNames=
LayoutList=gb
LayoutLoopCount=-1
Model=pc105
ResetOldOptions=false
ShowFlag=false
ShowLabel=true
ShowLayoutIndicator=true
ShowSingle=false
SwitchMode=Global
Use=true
VariantList=
EOF
```

Also set it in kwinrc:

```bash
kwriteconfig6 --file kwinrc --group Layout --key LayoutList gb
kwriteconfig6 --file kwinrc --group Layout --key Use 1
```

---

## 3. Environment variables (forces layout for Wayland)

Create a session environment file:

```bash
mkdir -p ~/.config/environment.d

cat > ~/.config/environment.d/keyboard.conf << 'EOF'
XKB_DEFAULT_LAYOUT=gb
XKB_DEFAULT_MODEL=pc105
XKB_DEFAULT_OPTIONS=
EOF
```

Or system-wide:

```bash
sudo tee /etc/environment.d/keyboard.conf << 'EOF'
XKB_DEFAULT_LAYOUT=gb
XKB_DEFAULT_MODEL=pc105
EOF
```

---

## 4. Console / TTY layout

```bash
sudo nano /etc/vconsole.conf
```

Set:

```
KEYMAP=uk
XKBLAYOUT=gb
```

---

## 5. Apply immediately (current session)

```bash
# May work for XWayland apps
setxkbmap gb

# For pure Wayland the most reliable way is to log out and back in
```

---

## 6. Verify

```bash
# Current XKB layout (if available)
setxkbmap -query

# localectl
localectl status

# Test the keys: try " (shift+2) and @ (shift+') – on UK they are swapped vs US
```

---

## 7. Optional – UK Extended (for accents)

If you need dead keys for Welsh / Gaelic / other accents:

```bash
sudo localectl set-x11-keymap gb pc105 ''
# or with variant if available:
# sudo localectl set-x11-keymap gb pc105 extd
```

Most users only need plain `gb`.

---

## 8. After changes

**Log out and log back in** (or reboot) so KWin picks up the new layout.

---

## Summary of files touched

| File | Purpose |
|------|---------|
| `localectl` | System X11 + console keymap |
| `~/.config/kxkbrc` | KDE / KWin layout list |
| `~/.config/environment.d/keyboard.conf` | Wayland XKB default |
| `/etc/vconsole.conf` | TTY / console |

---

*Works on Artix / Arch with LXQt + KWin Wayland when the GUI keyboard settings do not stick.*
