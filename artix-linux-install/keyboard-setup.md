# Set UK Keyboard Layout as Default – Artix OpenRC + LXQt + KWin

Simple guide for pure **OpenRC** (no systemd / no `localectl`).

UK layout code = **`gb`** (console keymap = **`uk`**)

---

## 1. Console / TTY

```bash
sudo nano /etc/vconsole.conf
```

Set the file to:

```
KEYMAP=uk
```

Save and exit.

---

## 2. Graphical session (KWin Wayland + XWayland)

```bash
mkdir -p ~/.config/environment.d

cat > ~/.config/environment.d/00-keyboard.conf << 'EOT'
XKB_DEFAULT_LAYOUT=gb
XKB_DEFAULT_MODEL=pc105
EOT
```

---

## 3. KWin layout config

```bash
cat > ~/.config/kxkbrc << 'EOT'
[Layout]
LayoutList=gb
Use=true
EOT
```

---

## 4. Apply

Log out and log back in (or reboot).

---

## 5. Test

After login, check that these keys produce UK symbols:

| Key          | Expected |
|--------------|----------|
| Shift + 2    | `"`      |
| Shift + '    | `@`      |
| `#` key      | `#`      |

Optional check:

```bash
setxkbmap -query 2>/dev/null || echo "setxkbmap not available (normal on pure Wayland)"
```

---

## Files changed

| File | Purpose |
|------|---------|
| `/etc/vconsole.conf` | TTY / console keymap |
| `~/.config/environment.d/00-keyboard.conf` | Wayland + XWayland default layout |
| `~/.config/kxkbrc` | KWin layout list |

---

*Artix OpenRC + LXQt + KWin Wayland – no systemd tools required.*
