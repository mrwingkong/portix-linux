# 09 – QTerminal on-screen keyboard (tablet mode)

Plasma Keyboard does not stay open in QTerminal. QTerminal is not a normal text field, so KWin hides the keyboard when you tap the terminal.

This guide adds a small watcher that:

- forces Plasma Keyboard up when you are in **tablet mode** and QTerminal is **focused**
- hides that forced keyboard when you click the desktop / another window, minimise QTerminal, close QTerminal, or fold back to **laptop mode**
- leaves Plasma Keyboard working as usual for browsers, notes, and other text fields

It does **not** change `07-wallpaper-orientation.md`. Leave the wallpaper script alone.

---

## Prerequisites (already done in 01 / 02)

These are normally required and were installed in the earlier stages:

- Packages: `plasma-keyboard`, `python-pyqt6`, `qt6-tools`
- KWin input method already set:

```bash
kreadconfig6 --file kwinrc --group Wayland --key InputMethod
```

That should print:

```text
/usr/share/applications/org.kde.plasma.keyboard.desktop
```

If any are missing:

```bash
sudo pacman -S --needed plasma-keyboard python-pyqt6 qt6-tools
kwriteconfig6 --file kwinrc --group Wayland --key InputMethod \
  "/usr/share/applications/org.kde.plasma.keyboard.desktop"
```

`qdbus6` comes from `qt6-tools`. The command is `qdbus6`, not `qdbus`.

---

## How tablet mode is detected

This machine uses the ThinkPad switch (not portrait vs landscape):

```text
/sys/devices/platform/thinkpad_acpi/hotkey_tablet_mode
```

- `0` = laptop
- `1` = tablet (any orientation)

Check:

```bash
cat /sys/devices/platform/thinkpad_acpi/hotkey_tablet_mode
```

---

## 1. Focus service

Writes the focused window to `/tmp/osk-focus-state`  
(`qterminal|0` = QTerminal focused and not minimised).

```bash
mkdir -p ~/.local/bin
```

```bash
cat > ~/.local/bin/osk-focus-service.py << 'EOF'
#!/usr/bin/env python3
import sys
from PyQt6.QtCore import QCoreApplication, QObject, pyqtSlot
from PyQt6.QtDBus import QDBusConnection

class Host(QObject):
    @pyqtSlot(str, str)
    def Set(self, cls, minimized):
        with open("/tmp/osk-focus-state", "w") as f:
            f.write(f"{cls}|{minimized}\n")

app = QCoreApplication(sys.argv)
host = Host()
bus = QDBusConnection.sessionBus()
if not bus.registerService("local.oskfocus"):
    sys.exit(0)
opts = QDBusConnection.RegisterOption.ExportAllSlots
if not bus.registerObject("/State", host, opts):
    sys.exit(1)
open("/tmp/osk-focus-state", "w").write("none|1\n")
app.exec()
EOF
chmod +x ~/.local/bin/osk-focus-service.py
```

---

## 2. KWin focus script

```bash
mkdir -p ~/.local/share/kwin-scripts
```

```bash
cat > ~/.local/share/kwin-scripts/osk-focus.js << 'EOF'
function report() {
    var w = workspace.activeWindow;
    var cls = "none";
    var min = "1";
    if (w) {
        cls = String(w.resourceClass);
        min = w.minimized ? "1" : "0";
        if (w.minimizedChanged)
            w.minimizedChanged.connect(report);
    }
    callDBus("local.oskfocus", "/State", "local.py.osk_focus_service.Host",
             "Set", cls, min);
}
workspace.windowActivated.connect(report);
workspace.windowAdded.connect(report);
workspace.windowRemoved.connect(report);
report();
EOF
```

---

## 3. Tablet watcher

```bash
cat > ~/.local/bin/osk-tablet-watch.sh << 'EOF'
#!/bin/bash
TABLET=/sys/devices/platform/thinkpad_acpi/hotkey_tablet_mode

is_tablet() { [ -f "$TABLET" ] && [ "$(cat "$TABLET")" = "1" ]; }

show() {
    qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.mode 2 >/dev/null 2>&1
    qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.forceActivate >/dev/null 2>&1
}
normal() {
    qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.active false >/dev/null 2>&1
    qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.mode 1 >/dev/null 2>&1
}

qterm_focused() {
    st=$(cat /tmp/osk-focus-state 2>/dev/null | tr '[:upper:]' '[:lower:]' | tr -d ' ')
    [ "$st" = "qterminal|0" ]
}

ensure_helpers() {
    pgrep -f osk-focus-service.py >/dev/null || \
        nohup /home/myname/.local/bin/osk-focus-service.py >/tmp/osk-focus-service.log 2>&1 &
    qdbus6 org.kde.KWin /Scripting org.kde.kwin.Scripting.unloadScript osk-focus >/dev/null 2>&1
    SID=$(qdbus6 org.kde.KWin /Scripting org.kde.kwin.Scripting.loadScript \
        /home/myname/.local/share/kwin-scripts/osk-focus.js osk-focus 2>/dev/null)
    [ -n "$SID" ] && qdbus6 org.kde.KWin /Scripting/Script$SID org.kde.kwin.Script.run >/dev/null 2>&1
}

ensure_helpers

forced=0
while true; do
    if is_tablet && qterm_focused; then
        forced=1
        vis=$(qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.visible 2>/dev/null)
        [ "$vis" != "true" ] && show
    else
        if [ "$forced" -eq 1 ]; then
            normal
            forced=0
        fi
    fi
    sleep 0.4
done
EOF
chmod +x ~/.local/bin/osk-tablet-watch.sh
```

Change `myname` in that script to your username before using it.

Do **not** start this with a plain `&` inside QTerminal. Closing that terminal can kill the watcher. Use `nohup` as below, or autostart.

---

## 4. Autostart

```bash
mkdir -p ~/.config/autostart
```

```bash
cat > ~/.config/autostart/osk-focus-service.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=OSK Focus Service
Exec=/home/myname/.local/bin/osk-focus-service.py
X-GNOME-Autostart-enabled=true
OnlyShowIn=LXQt;
EOF
```

```bash
cat > ~/.config/autostart/osk-tablet-watch.desktop << 'EOF'
[Desktop Entry]
Type=Application
Name=OSK Tablet Watch
Exec=/usr/bin/bash -c "nohup /home/myname/.local/bin/osk-tablet-watch.sh >/tmp/osk-watch.log 2>&1 &"
X-GNOME-Autostart-enabled=true
OnlyShowIn=LXQt;
EOF
```

Change `myname` in both files.

---

## 5. Start now (this session)

```bash
pkill -f osk-focus-service.py 2>/dev/null
pkill -f osk-tablet-watch.sh 2>/dev/null
nohup ~/.local/bin/osk-focus-service.py >/tmp/osk-focus-service.log 2>&1 &
disown
sleep 0.3
nohup ~/.local/bin/osk-tablet-watch.sh >/tmp/osk-watch.log 2>&1 &
disown
```

```bash
pgrep -af osk-focus-service.py
pgrep -af osk-tablet-watch
```

Both lines must show a running process.

Load the KWin script once for this session:

```bash
qdbus6 org.kde.KWin /Scripting org.kde.kwin.Scripting.unloadScript osk-focus
SID=$(qdbus6 org.kde.KWin /Scripting org.kde.kwin.Scripting.loadScript "$HOME/.local/share/kwin-scripts/osk-focus.js" osk-focus)
qdbus6 org.kde.KWin /Scripting/Script$SID org.kde.kwin.Script.run
```

---

## 6. Test

Focus check (laptop is fine):

```bash
cat /tmp/osk-focus-state
```

With QTerminal focused you should see `qterminal|0`.

```bash
sleep 6; cat /tmp/osk-focus-state
```

Press Enter, then click the desktop. You should see something like `pcmanfm-qt|0`.

Tablet checks:

| Action | Keyboard |
|--------|----------|
| Fold to tablet, tap QTerminal | Forced keyboard on |
| Tap desktop or another window | Forced keyboard off (normal IM for that app) |
| Minimise QTerminal | Forced keyboard off |
| Restore / tap QTerminal | Forced keyboard on |
| Close QTerminal | Forced keyboard off |
| Fold back to laptop | Forced keyboard off |

Use the **normal** QTerminal shortcut. No extra tablet launcher is required.

---

## 7. Troubleshooting

| Symptom | Likely cause | Fix |
|---------|--------------|-----|
| Nothing in tablet mode | Watcher died (started with `&` in QTerminal) | Start with `nohup` + `disown`, check autostart |
| `pgrep -af osk-tablet-watch` empty | Watcher not running | Run section 5 again |
| State stays `none\|1` | Focus service or KWin script not loaded | Restart service + `loadScript` / `run` |
| Keyboard stuck after laptop mode | Mode left at `2` | `qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.mode 1` |
| `qdbus: command not found` | Wrong binary | Use `qdbus6` from `qt6-tools` |
| Tablet file missing | Not this ThinkPad path | Check `find /sys -iname '*tablet*'` |

Useful commands:

```bash
cat /sys/devices/platform/thinkpad_acpi/hotkey_tablet_mode
cat /tmp/osk-focus-state
pgrep -af osk-tablet-watch
pgrep -af osk-focus-service.py
pgrep -ax qterminal
qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.mode
qdbus6 org.kde.KWin /VirtualKeyboard org.kde.kwin.VirtualKeyboard.visible
```

---

## 8. What this does not change

- Wallpaper / panel recover (`07`)
- SwayNC volume / brightness
- Biometrics / PAM
- The normal QTerminal `.desktop` name — only behaviour while the watcher runs

---

*Artix OpenRC only – no systemd.*
