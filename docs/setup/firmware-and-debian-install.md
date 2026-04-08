# Firmware Flash and Debian Install — Cyan (Acer R11 CB5-132T)

## Hardware

- **Model:** Acer Chromebook R11 CB5-132T
- **Board codename:** CYAN
- **CPU:** Intel Celeron N3160 (Braswell, 4-core)
- **RAM:** 4GB
- **Storage:** 16GB eMMC (internal, `/dev/mmcblk0`)
- **Display:** 1366x768 touchscreen

---

## The Problem: Ctrl+L and EOL Blocking

Out of the box, Chromebooks run ChromeOS with verified boot — only Google-signed OS images can boot. To run Linux you have two options:

- **Ctrl+L (legacy boot / SeaBIOS):** Press Ctrl+L at the developer mode splash screen to invoke a basic legacy BIOS. Limited, no UEFI, no sleep/suspend support.
- **Full UEFI (MrChromebox):** Completely replaces the Chromebook firmware with coreboot + UEFI. Full Linux support, proper ACPI, sleep, GRUB, etc.

We wanted full UEFI. The problem: the CYAN board is **End of Life (EOL)**, and the MrChromebox firmware utility script checks for this and refuses to flash EOL boards by default.

**The bypass:** The script has a variable `isEOL` for Braswell (BSW) boards. By patching the script before running it (setting `BSW isEOL=false`), the EOL check is skipped and the flash proceeds normally.

---

## Step 1: Enable Developer Mode

1. Hold **Esc + Refresh (F3) + Power** to enter recovery mode.
2. At the recovery screen press **Ctrl+D**.
3. Confirm to enable developer mode — the Chromebook wipes itself and reboots.
4. At the "OS verification is OFF" splash screen, press **Ctrl+D** each boot (or wait 30 seconds).

---

## Step 2: Disable Firmware Write Protection

Chromebooks have hardware write protection on the firmware chip. On the R11 this is controlled by the battery connection — disconnecting the battery briefly disables WP.

1. Open the back of the Chromebook (Phillips screws around the edge).
2. Disconnect the battery connector from the motherboard.
3. Hold the power button for 5 seconds to drain residual charge.
4. Reconnect the battery.

Write protection is now disabled until the next time you fully power cycle with the battery connected at boot.

---

## Step 3: Flash Full UEFI Firmware

Boot ChromeOS, open the terminal (Ctrl+Alt+T → `shell`), then:

```bash
# Download MrChromebox firmware utility
curl -LO https://mrchromebox.tech/firmware-util.sh

# Patch out the EOL block for Braswell boards
sed -i 's/isEOL=true/isEOL=false/' firmware-util.sh

# Run it
sudo bash firmware-util.sh
```

In the menu, choose **option 2: Flash Full ROM (UEFI)**.

- The script downloads and flashes coreboot with a TianoCore UEFI payload.
- It backs up the original firmware first — **save this backup somewhere safe.**
  - Our backup is on ray at: `~/BACKUP-CYAN-Google_Unknown-2026.04.07.rom`

After flashing, the Chromebook reboots into a standard UEFI environment. No more ChromeOS splash, no Ctrl+L needed.

---

## Step 4: Install Debian 13 (Trixie)

### Boot the installer

1. Write the Debian netinstall ISO to a USB drive.
2. Plug it in, power on — UEFI picks it up automatically (or press Escape for the boot menu).
3. Boot the Debian installer.

### Installer walkthrough

The installer is mostly standard, but a few things came up on this hardware:

**Language / hostname:** Standard, fill in as prompted.

**Root filesystem device:** The installer asked which device to use as the default root filesystem. The internal eMMC shows up as:

- `/dev/mmcblk0` — the whole eMMC
- `/dev/mmcblk0boot0` — boot partition 0 (don't use)
- `/dev/mmcblk0boot1` — boot partition 1 (don't use)
- `/dev/mmcblk0p1` — the actual main partition ✓

Use `/dev/mmcblk0p1`. The `boot0`/`boot1` devices are hardware partitions for the eMMC controller, not regular partitions.

**Errors during install:** A few non-fatal errors appeared (`modinfo.sh does not exist`, some firmware fetch warnings). These are cosmetic — the install completes fine.

**Bootloader:** GRUB installs to the eMMC. After install, the system boots straight to GRUB then Debian — no ChromeOS, no UEFI splash.

### Post-install GRUB config

GRUB was set to skip the menu (timeout=0, hidden) so it boots straight to Debian:

```bash
# /etc/default/grub
GRUB_TIMEOUT=0
GRUB_TIMEOUT_STYLE=hidden
```

```bash
sudo update-grub
```

---

## Step 5: Essential Packages

```bash
sudo apt update && sudo apt install -y \
  firmware-linux firmware-linux-nonfree intel-microcode \
  libavcodec-extra ffmpeg \
  openssh-server \
  tailscale \
  brightnessctl \
  pulseaudio-utils \
  network-manager
```

**Firmware packages** are important — without them WiFi and graphics acceleration may not work on Braswell hardware.

---

## Step 6: Network

### WiFi
NetworkManager handles WiFi. Use `nmtui` in the terminal for a simple UI.

### SSH
```bash
sudo systemctl enable --now ssh
```

SSH in from another machine: `ssh andrew@<cyan-ip>`

### Tailscale
```bash
curl -fsSL https://tailscale.com/install.sh | sh
sudo tailscale up
```

After authenticating, cyan gets a stable Tailscale IP (`100.118.138.19`) that works across networks.

The SSH alias on ray (`~/.ssh/config`):
```
Host cyan
    HostName 100.118.138.19
    User andrew
    IdentityFile ~/.ssh/id_rsa
```

---

## Autologin to TTY

To boot straight into i3 without a login screen:

```bash
sudo mkdir -p /etc/systemd/system/getty@tty1.service.d
sudo tee /etc/systemd/system/getty@tty1.service.d/autologin.conf << EOF
[Service]
ExecStart=
ExecStart=-/sbin/agetty --autologin andrew --noclear %I \$TERM
EOF
```

Then in `~/.bash_profile`:
```bash
if [ "$(tty)" = "/dev/tty1" ]; then
    exec startx
fi
```

And `~/.xinitrc`:
```bash
exec i3
```

On boot: getty autologins → bash runs `.bash_profile` → `startx` → i3 launches.
