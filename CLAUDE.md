# CYAN — Machine Context for Claude

This file is loaded automatically by Claude Code. It describes the cyan machine, its setup, and how to work with it.

---

## What is Cyan?

Cyan is an **Acer Chromebook R11 CB5-132T** running **Debian 13 (Trixie)** with a full UEFI coreboot firmware replacement. It is Andrew's portable/secondary machine, used alongside **ray** (the main desktop server).

---

## Hardware

| Component | Details |
|-----------|---------|
| Model | Acer Chromebook R11 CB5-132T |
| Board codename | CYAN (Intel Braswell) |
| CPU | Intel Celeron N3060 @ 1.60GHz (2-core) |
| RAM | 4GB |
| Storage | 16GB eMMC — root at `/dev/mmcblk0p2`, ~6.4GB free |
| Display | 1366x768 touchscreen (touchscreen currently non-functional on Linux) |
| GPU | Intel Atom/Celeron integrated (i915) |
| WiFi | Intel (requires firmware-linux-nonfree) |
| Keyboard | Chromebook layout — Search key = Mod/Super |

---

## Firmware

- **Original:** Google coreboot (ChromeOS)
- **Current:** MrChromebox full UEFI (coreboot + TianoCore) — flashed with EOL bypass (`BSW isEOL=false` patch)
- **Bootloader:** GRUB (hidden, timeout=0, boots straight to Debian)
- Firmware backup stored on ray

---

## OS

- **Distro:** Debian 13 (Trixie)
- **Kernel:** 6.12.74+deb13+1-amd64
- **Login:** Autologin to TTY1 via getty → startx → i3

---

## Network / SSH

| | Value |
|-|-------|
| Local IP | 10.45.69.86 (may vary) |
| Tailscale IP | 100.118.138.19 (stable) |
| SSH user | andrew |
| SSH from ray | `ssh cyan` (configured in ~/.ssh/config) |

Tailscale is installed — use the Tailscale IP for reliable cross-network access.

---

## Desktop Environment

**i3** window manager with **polybar** status bar.

- Config: `~/.config/i3/config`
- Mod key: Search key (Mod4)
- Top-row keys send F6–F10 (coreboot UEFI behaviour, not XF86 keysyms)

### Polybar

- Config: `~/.config/polybar/config.ini`
- Scripts in `~/.config/polybar/`:
  - `weather.sh` — Open-Meteo API, Etobicoke ON (lat=43.64, lon=-79.57), Symbols Nerd Font icons
  - `network.sh` — WiFi signal dots from `/proc/net/wireless`
  - `battery.sh` — Custom thresholds: 75/50/25/8%, dots using font-1 (size=12)
  - `date.sh` — Date and time string
- Restart polybar: `pkill polybar; polybar main & disown`

### Terminal

**Alacritty** — config at `~/.config/alacritty/alacritty.toml`
- Font: DejaVu Sans Mono 9pt
- Background: `#1e1b2e` (purple theme)

### Theme

Purple — colours:
- Background: `#1e1b2e`
- Accent/focused: `#7c3aed`
- Purple highlight: `#bd93f9`

---

## Key Software

| Package | Purpose |
|---------|---------|
| i3 | Window manager |
| polybar | Status bar |
| alacritty | Terminal |
| rofi | App launcher (Mod+Space) |
| dunst | Notifications |
| picom | Compositor (transparency) |
| redshift-gtk | Blue light filter |
| flameshot | Screenshots (Print key) |
| nm-applet / nmtui | Network management |
| brightnessctl | Brightness control |
| Symbols Nerd Font | Weather icons in polybar |
| FontAwesome | Fallback icon font |
| Tailscale | VPN / cross-network SSH |

---

## Autostart (on i3 launch)

- `nm-applet` — WiFi tray
- `picom -b` — compositor
- `dunst` — notifications
- `redshift-gtk` — blue light
- `polybar main` — status bar
- `alacritty -e ssh ray` — auto-SSH into ray on boot

---

## Known Issues

### Touchscreen not working
- Device: `i2c-ELAN0001:00` (detected on I2C bus 0)
- Driver `elants_i2c` is loaded but probe fails — suspected ACPI power sequencing issue with coreboot UEFI
- `chromeos_laptop` module fails to load ("No such device")
- **Status: In progress** — needs DSDT patch or kernel workaround

---

## Related Machines

### ray
- Main desktop/server
- SSH: `ssh ray` from cyan, or via Tailscale
- Runs upload server on port 8080 (`/tmp/uploads/`)
- Most heavy compute happens here

### zero
- Another machine on the network
- ARM-based (eMMC root at `/dev/mmcblk0p1`)
- Used for USB formatting tasks

---

## Project Layout

```
~/projects/
  cyan/           ← this repo (configs, docs)
    docs/
      i3-keybindings.md
      setup/
        firmware-and-debian-install.md
        i3-setup-and-customization.md
  games/
    wolfenstein/
    rollercoastertycoon/
```

---

## Working Style Notes

- Andrew prefers concise responses — skip summaries of what was just done
- Changes to cyan are made via SSH from ray: `ssh cyan "..."`
- Config files are often fetched to ray with `scp`, edited locally, then pushed back
- Use `& disown` (not just `&`) for background GUI processes launched over SSH
- Keybindings go in i3 config, not xbindkeys (xbindkeys conflicts with Firefox)
