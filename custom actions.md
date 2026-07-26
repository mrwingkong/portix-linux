# LXQt + Labwc + Swaync Setup Guide

Complete working setup for:

- Clean volume & brightness OSDs (with progress bar)
- Battery low warnings + Ignore button
- Auto-closing sticky WiFi & USB notifications
- HDR toggle (with clean exit to TTY)
- Smart brightness (real backlight in SDR, gamma in HDR)

---

## 1. Install Packages

```bash
sudo pacman -S swaync brightnessctl wireplumber alsa-utils bc glib2-devel base-devel wayland wlroots
