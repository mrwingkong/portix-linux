# Fingerprint Reader Setup – Artix OpenRC + LXQt

Simple guide for pure **OpenRC** (no systemd).

---

## 1. Install

```bash
sudo pacman -S fprintd
```

---

## 2. Start the daemon

There is no default OpenRC service for fprintd. Start it directly:

```bash
# Find the binary
find /usr -name 'fprintd' 2>/dev/null

# Start it (most common path)
sudo /usr/lib/fprintd &
```

### Make it start on boot

Create a simple OpenRC service:

```bash
sudo tee /etc/init.d/fprintd << 'EOT'
#!/sbin/openrc-run

name="fprintd"
command="/usr/lib/fprintd"
command_background=true
pidfile="/run/fprintd.pid"
EOT

sudo chmod +x /etc/init.d/fprintd
sudo rc-update add fprintd default
sudo rc-service fprintd start
```

If the binary is not at `/usr/lib/fprintd`, change the `command=` line to the path returned by `find`.

Check it is running:

```bash
ps aux | grep fprintd | grep -v grep
```

---

## 3. Available fingers

| Name                    | Finger            |
|-------------------------|-------------------|
| `left-thumb`            | Left thumb        |
| `left-index-finger`     | Left index        |
| `left-middle-finger`    | Left middle       |
| `left-ring-finger`      | Left ring         |
| `left-little-finger`    | Left little       |
| `right-thumb`           | Right thumb       |
| `right-index-finger`    | Right index (default) |
| `right-middle-finger`   | Right middle      |
| `right-ring-finger`     | Right ring        |
| `right-little-finger`   | Right little      |

---

## 4. Enroll fingerprints

### Default (right index finger)

```bash
fprintd-enroll
```

### Specific finger

```bash
fprintd-enroll -f left-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f left-thumb
```

Place/swipe the finger when prompted until you see `enroll-completed`.

### Recommended set (both hands)

```bash
fprintd-enroll -f right-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f left-index-finger
fprintd-enroll -f left-thumb
```

### List and test

```bash
fprintd-list "$USER"
fprintd-verify
```

### Delete all and start again

```bash
fprintd-delete "$USER"
```

---

## 5. Enable fingerprint for login / sudo / polkit

### Login

```bash
sudo nano /etc/pam.d/system-local-login
```

Add this line **at the very top** of the file (before other `auth` lines):

```
auth      sufficient    pam_fprintd.so
```

### sudo

```bash
sudo nano /etc/pam.d/sudo
```

Add at the top:

```
auth      sufficient    pam_fprintd.so
```

### Graphical password prompts (polkit)

```bash
sudo nano /etc/pam.d/polkit-1
```

Add at the top:

```
auth      sufficient    pam_fprintd.so
```

`sufficient` means: if the fingerprint succeeds, access is granted; if it fails, the normal password prompt is still used.

---

## 6. Test

```bash
# Should ask for fingerprint
sudo -k
sudo true

# Or log out and try logging in with your finger
```
##Fingerprint at login (graphical + TTY) only covers the login PAM stack.
Inside an already-logged-in session, the terminal uses different PAM files — mainly sudo (and sometimes su / polkit).
Make fingerprint work for sudo in the terminal
```
sudo -k          # clear cached credentials
sudo true        # should ask for fingerprint, not only password
```
If it’s missing, add it:
```
sudo nano /etc/pam.d/sudo
```
Put this at the very top of the file (first auth line)
```
auth      sufficient    pam_fprintd.so
```
sudo -k          # clear cached credentials
sudo true        # should ask for fingerprint, not only password



---

## Files / commands summary

| Item | Purpose |
|------|---------|
| `fprintd` package | Driver + tools |
| `/etc/init.d/fprintd` | OpenRC service (created by you) |
| `fprintd-enroll -f <finger>` | Enroll a finger |
| `/etc/pam.d/system-local-login` | Login with fingerprint |
| `/etc/pam.d/sudo` | sudo with fingerprint |
| `/etc/pam.d/polkit-1` | GUI auth with fingerprint |

---

*Artix OpenRC + LXQt – no systemd required.*
