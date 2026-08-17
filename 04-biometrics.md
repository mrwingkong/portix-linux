# 04 – Biometrics (Fingerprint)

Fingerprint reader setup for Artix OpenRC + LXQt + KWin.

---

## Prerequisites (already done in 01 / 02)

These packages and groups are normally required and were installed in the base/post stages:

- `fprintd` (installed in 02-post-install.md)
- User `myname` exists and is in the `wheel` group

If you skipped earlier steps, install now:

```bash
sudo pacman -S --needed fprintd
```

---

## 1. OpenRC service for fprintd

Artix does not ship a default OpenRC service for fprintd. Create one:

```bash
sudo tee /etc/init.d/fprintd > /dev/null << 'EOF'
#!/sbin/openrc-run

name="fprintd"
command="/usr/lib/fprintd"
command_background=true
pidfile="/run/fprintd.pid"
EOF
```

```bash
sudo chmod +x /etc/init.d/fprintd
sudo rc-update add fprintd default
sudo rc-service fprintd start
```

If the binary is not at `/usr/lib/fprintd`, locate it first:

```bash
find /usr -name 'fprintd' 2>/dev/null
```

Then edit the `command=` line in the service file.

Verify:

```bash
ps aux | grep fprintd | grep -v grep
rc-service fprintd status
```

---

## 2. Available fingers

| Name                    | Finger            |
|-------------------------|-------------------|
| left-thumb              | Left thumb        |
| left-index-finger       | Left index        |
| left-middle-finger      | Left middle       |
| left-ring-finger        | Left ring         |
| left-little-finger      | Left little       |
| right-thumb             | Right thumb       |
| right-index-finger      | Right index (default) |
| right-middle-finger     | Right middle      |
| right-ring-finger       | Right ring        |
| right-little-finger     | Right little      |

---

## 3. Enroll fingerprints

Default (right index):

```bash
fprintd-enroll
```

Specific finger:

```bash
fprintd-enroll -f left-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f left-thumb
```

Recommended set:

```bash
fprintd-enroll -f right-index-finger
fprintd-enroll -f right-thumb
fprintd-enroll -f left-index-finger
fprintd-enroll -f left-thumb
```

List and test:

```bash
fprintd-list "$USER"
fprintd-verify
```

Delete all and start over:

```bash
fprintd-delete "$USER"
```

---

## 4. Enable fingerprint for login and sudo

### Login

```bash
sudo nano /etc/pam.d/system-local-login
```

Add this line **at the very top** (before other `auth` lines):

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

`sufficient` means: fingerprint success grants access; failure falls back to password.

---

## 5. Important – Do NOT create a one-line polkit file

**Do not** create or replace `/etc/pam.d/polkit-1` with only the fprintd line.

A file that contains only:

```
auth      sufficient    pam_fprintd.so
```

breaks `pkexec`. That causes the LXQt backlight panel widget to crash the entire panel when clicked.

If you already created that file, fix it with:

```bash
sudo mv /etc/pam.d/polkit-1 /etc/pam.d/polkit-1.fingerprint-backup
```

Then test:

```bash
pkexec true
```

It should ask for your password (or succeed) instead of failing with “Not authorized”.

Fingerprint for graphical polkit prompts is optional and can be left out. Login and sudo fingerprint still work without touching polkit.

---

## 6. Test

```bash
sudo -k
sudo true
```

You should be prompted for a fingerprint.  
Also try logging out and logging back in with your finger.

---

## Files / commands summary

| Item                            | Purpose                        |
|---------------------------------|--------------------------------|
| `fprintd` package               | Driver + tools                 |
| `/etc/init.d/fprintd`           | OpenRC service (created above) |
| `fprintd-enroll -f <finger>`    | Enroll a finger                |
| `/etc/pam.d/system-local-login` | Login with fingerprint         |
| `/etc/pam.d/sudo`               | sudo with fingerprint          |
| `/etc/pam.d/polkit-1`           | **Do not create a one-line version** |

---

*Artix OpenRC only – no systemd.*
