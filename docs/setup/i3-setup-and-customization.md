# i3 Setup and Customization — Cyan

## What is i3?

i3 is a **tiling window manager** — instead of dragging windows around like a normal desktop, it automatically arranges them in a grid. Every window gets a slice of the screen. No mouse needed for most things.

It uses almost no RAM and has no background services — ideal for a low-power Chromebook.

---

## Packages Installed

```bash
sudo apt install -y \
  i3 \
  polybar \
  alacritty \
  rofi \
  dunst \
  picom \
  redshift-gtk \
  flameshot \
  nm-applet \
  brightnessctl \
  fonts-dejavu \
  fonts-font-awesome
```

**Symbols Nerd Font** was installed manually (used for weather icons in polybar):
```bash
mkdir -p ~/.local/share/fonts
# Download Symbols Nerd Font and copy .ttf files to above directory
fc-cache -fv
```

---

## i3 Config (`~/.config/i3/config`)

### Mod key
The **Mod key is the Search key** (magnifying glass, top-left of keyboard row). This is `Mod4` in i3.

```
set $mod Mod4
```

### Font
```
font pango:DejaVu Sans Mono 11
```

### Window borders and gaps
```
default_border pixel 2
gaps inner 6
gaps outer 3
```
`pixel 2` means a 2px border with no title bar. Gaps add breathing room between windows.

### Colors (purple theme)
```
client.focused          #7c3aed #7c3aed #ffffff #bd93f9 #7c3aed
client.unfocused        #2d1b3d #2d1b3d #888888 #2d1b3d #2d1b3d
client.focused_inactive #3d2b4d #3d2b4d #aaaaaa #3d2b4d #3d2b4d
```

### Autostart
```
exec --no-startup-id nm-applet        # WiFi tray icon
exec --no-startup-id picom -b        # Compositor (transparency)
exec --no-startup-id dunst           # Notification daemon
exec --no-startup-id redshift-gtk    # Blue light filter
exec --no-startup-id polybar main &  # Status bar
exec --no-startup-id alacritty -e ssh ray  # Auto-SSH into ray on boot
```

### Chromebook top-row key mapping
With coreboot UEFI firmware, the Chromebook top-row keys send **F6–F10**, not the XF86 media keysyms that Linux normally expects. Mapped manually in i3:

```
bindsym F6  exec brightnessctl set 10%-   # Brightness down
bindsym F7  exec brightnessctl set +10%   # Brightness up
bindsym F8  exec pactl set-sink-mute @DEFAULT_SINK@ toggle  # Mute
bindsym F9  exec pactl set-sink-volume @DEFAULT_SINK@ -5%   # Volume down
bindsym F10 exec pactl set-sink-volume @DEFAULT_SINK@ +5%   # Volume up
```

Each also fires a `notify-send` so dunst shows a brief on-screen notification.

Note: F1 and F2 (Back/Forward keys) are used for window focus navigation, so brightness/volume start at F6.

**Why not xbindkeys?** We tried xbindkeys first but it conflicted with Firefox — the keys would trigger both the i3 binding and browser actions. Moving everything into i3 config fixed it.

---

## Alacritty (`~/.config/alacritty/alacritty.toml`)

Alacritty is the terminal emulator — GPU-accelerated, fast, minimal.

```toml
[window]
padding = { x = 8, y = 8 }
opacity = 0.95

[font]
size = 9.0

[font.normal]
family = "DejaVu Sans Mono"

[cursor]
style = { shape = "Block", blinking = "Always" }
blink_interval = 500

[colors.primary]
background = "#1e1b2e"
foreground = "#e0e0e0"

[env]
TERM = "xterm-256color"
```

The background `#1e1b2e` matches the polybar background for a unified look.

---

## Polybar (`~/.config/polybar/config.ini`)

Polybar is the status bar at the top of the screen.

### Fonts
Three fonts are defined — polybar uses 1-based indexing for font tags (`%{T1}`, `%{T2}`, `%{T3}`):

```ini
font-0 = DejaVu Sans Mono:size=10;2    # T1 — main text, vertically centered
font-1 = DejaVu Sans Mono:size=12;1   # T2 — dots (battery/network), slightly larger
font-2 = Symbols Nerd Font:size=15;4  # T3 — weather icons, larger with lower offset
```

The `;N` at the end is the **vertical offset** — positive moves text down, used to vertically align icons that sit too high or low.

### Modules (left to right)
- **i3** — workspace numbers, focused workspace highlighted in purple
- **+** (newterm) — click to open a new terminal
- **weather** — current conditions, icon + temperature
- **network** — WiFi signal as dots
- **battery** — charge level as dots + percentage
- **date** — day, date, time

---

## Polybar Scripts

### Weather (`~/.config/polybar/weather.sh`)

Uses the **Open-Meteo API** (free, no key needed) for Etobicoke, ON:

```
https://api.open-meteo.com/v1/forecast?latitude=43.64&longitude=-79.57&current=temperature_2m,weather_code&timezone=America/Toronto
```

The weather code maps to a Symbols Nerd Font icon (e.g., `\ue30d` = sun, `\ue30c` = partly cloudy). The icon is wrapped in `%{T3}` to use the Nerd Font and colored based on conditions. Temperature is always yellow.

**Why Open-Meteo?** Environment Canada's XML feed returned 404. Open-Meteo is free, reliable, and requires no API key.

Updates every 600 seconds (10 minutes).

### Network (`~/.config/polybar/network.sh`)

Reads WiFi signal strength from `/proc/net/wireless` and outputs colored dots:

| Signal | Dots | Color |
|--------|------|-------|
| ≥ 75 | ●●●● | Blue |
| ≥ 50 | ○●●● | Blue |
| ≥ 25 | ○○●● | Yellow |
| ≥ 10 | ○○○● | Red |
| < 10 | ○○○○ | Red |

Click opens `nmtui` (network manager TUI) in a new terminal.

**Important:** Use `alacritty -e nmtui &` with `&` so the bar doesn't freeze waiting for the terminal to close. And use `& disown` when restarting polybar manually so it doesn't die when the terminal closes.

### Battery (`~/.config/polybar/battery.sh`)

Custom script (instead of polybar's internal battery module) to support custom percentage thresholds.

Reads from `/sys/class/power_supply/BAT0/capacity` and `/sys/class/power_supply/BAT0/status`.

| Percentage | Dots | Color |
|------------|------|-------|
| ≥ 75% | ●●●● | Green |
| ≥ 50% | ○●●● | Green |
| ≥ 25% | ○○●● | Green |
| ≥ 8% | ○○○● | Red |
| < 8% | ○○○○ | Red |

Uses `%{T2}` (font-1, size=12) so the dots match the network dots in size and vertical position.

Click opens a terminal with `upower` battery details.

**Why custom script?** Polybar's internal battery module calculates ramp thresholds evenly (every 20%), and the `ramp-capacity` token can't be used in `format-charging`. The custom script gives full control over both thresholds and charging display.

### Date (`~/.config/polybar/date.sh`)

```bash
date +'%a %b %d  %I:%M %p'
```

Output example: `Tue Apr 07  02:34 PM`

---

## Purple Theme

Colors used across all three configs:

| Role | Color |
|------|-------|
| Background | `#1e1b2e` |
| Focused border | `#7c3aed` |
| Purple accent | `#bd93f9` |
| Unfocused bg | `#2d1b3d` |
| Green (battery/network good) | `#50fa7b` |
| Red (battery/network bad) | `#ff5555` |
| Yellow (weather text) | `#f1fa8c` |
| Gray (unfocused workspaces) | `#888888` |

---

## Troubleshooting

**Polybar closes when terminal closes:**
Use `& disown` instead of just `&`:
```bash
pkill polybar; polybar main & disown
```

**Keys triggering things in Firefox:**
Don't use xbindkeys — put all keybindings in the i3 config instead.

**Keyboard layout wrong (slash key broken):**
Was accidentally set to Canadian French. Fix:
```bash
setxkbmap us
```
To make permanent, add to `~/.xinitrc` before `exec i3`.

**SSH host key error after reinstall:**
```bash
ssh-keygen -R cyan  # or the IP
```

**Polybar date showing `%date% %time%` literally:**
Use a `custom/script` module with `date` command instead of `internal/date`.
