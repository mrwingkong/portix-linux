# Fingerprint Reader Setup (fprintd) – Artix / Arch + LXQt

Complete guide for enrolling multiple fingers and enabling fingerprint authentication for login, sudo, and polkit.

---

## 1. Install packages

```bash
sudo pacman -S fprintd
```

Optional (if your reader needs extra drivers):

```bash
# Only if your device is not detected
# Check: lsusb | grep -i finger
```

Start the daemon (Artix has no OpenRC service by default):

```bash
sudo /usr/lib/fprintd &
# or
sudo systemctl start fprintd   # if you use a systemd compatibility layer
```

---

## 2. Check the reader is detected

```bash
fprintd-list "$USER"
lsusb | grep -iE 'finger|synaptics|goodix|elan|validity'
```

---

## 3. Available fingers

| Finger name              | Description        |
|--------------------------|--------------------|
| `left-thumb`             | Left thumb         |
| `left-index-finger`      | Left index         |
| `left-middle-finger`     | Left middle        |
| `left-ring-finger`       | Left ring          |
| `left-little-finger`     | Left little        |
| `right-thumb`            | Right thumb        |
| `right-index-finger`     | Right index (default) |
| `right-middle-finger`    | Right middle       |
| `right-ring-finger`      | Right ring         |
| `right-little-finger`    | Right little       |

---

## 4. Enroll fingerprints

### Single finger (default = right index)

```bash
fprintd-enroll
```

### Specific finger

```bash
fprintd-enroll -f left-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f left-thumb
```

You will be asked to place/swipe the finger several times until you see `enroll-completed`.

### Enroll several useful fingers (recommended)

```bash
# Right hand
fprintd-enroll -f right-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f right-middle-finger

# Left hand (backup)
fprintd-enroll -f left-index-finger
fprintd-enroll -f left-thumb
```

### Enroll every finger (optional)

```bash
for finger in left-thumb left-index-finger left-middle-finger left-ring-finger left-little-finger \
              right-thumb right-index-finger right-middle-finger right-ring-finger right-little-finger; do
    echo "=== Enrolling $finger ==="
    fprintd-enroll -f "$finger"
done
```

---

## 5. List and verify

```bash
# List enrolled fingers
fprintd-list "$USER"

# Test verification
fprintd-verify
```

---

## 6. Enable fingerprint in PAM

### Login / session

Edit (as root):

```bash
sudo nano /etc/pam.d/system-local-login
```

Add this line **at the very top** of the `auth` section (before any other `auth` lines):

```
auth      sufficient    pam_fprintd.so
```

Example:

```
#%PAM-1.0
auth      sufficient    pam_fprintd.so
auth      include       system-login
...
```

### sudo

```bash
sudo nano /etc/pam.d/sudo
```

Add at the top of the auth section:

```
auth      sufficient    pam_fprintd.so
```

### polkit (graphical password prompts)

```bash
sudo nano /etc/pam.d/polkit-1
```

Add at the top:

```
auth      sufficient    pam_fprintd.so
```

> **Note:** `sufficient` means fingerprint success grants access immediately. If the finger fails or is not enrolled, it falls through to the normal password.

---

## 7. Delete fingerprints

```bash
# Delete all fingerprints for the current user
fprintd-delete "$USER"

# Then re-enroll as needed
```

---

## 8. Make fprintd start on boot (Artix)

Create a simple OpenRC service:

```bash
sudo tee /etc/init.d/fprintd << 'EOF'
#!/sbin/openrc-run
command="/usr/lib/fprintd"
command_background=true
pidfile="/run/fprintd.pid"
EOF

sudo chmod +x /etc/init.d/fprintd
sudo rc-update add fprintd default
sudo rc-service fprintd start
```

If the binary path is different:

```bash
find /usr -name 'fprintd' 2>/dev/null
```

---

## 9. Tips

- Enroll at least one finger from each hand.
- Wet, dirty, or injured fingers may fail — keep a password fallback.
- Some readers only support swipe; others support press. Follow the on-screen prompts.
- After PAM changes, a new login session is required for fingerprint login to appear.

---

*Tested for Artix / Arch + LXQt + KWin Wayland setups.*
