<div align="center">

```
██████╗ ██╗████████╗ █████╗ ██╗   ██╗██████╗
██╔══██╗██║╚══██╔══╝██╔══██╗██║   ██║██╔══██╗
██████╔╝██║   ██║   ███████║██║   ██║██║  ██║
██╔══██╗██║   ██║   ██╔══██║██║   ██║██║  ██║
██████╔╝██║   ██║   ██║  ██║╚██████╔╝██████╔╝
╚═════╝ ╚═╝   ╚═╝   ╚═╝  ╚═╝ ╚═════╝ ╚═════╝
```

**Dual audio output router for Arch Linux + Hyprland**

Play audio on two devices simultaneously — AUX, Bluetooth, USB, HDMI — any combination.

![Arch Linux](https://img.shields.io/badge/Arch_Linux-1793D1?style=flat&logo=arch-linux&logoColor=white)
![PipeWire](https://img.shields.io/badge/PipeWire-native-orange?style=flat)
![Shell](https://img.shields.io/badge/Shell-Bash-green?style=flat)
![License](https://img.shields.io/badge/License-MIT-blue?style=flat)

</div>

---

## What is bitaud?

`bitaud` is a CLI tool that lets you route audio to **two output devices at the same time** on Arch Linux with PipeWire. Pick any two devices — wired headphones, Bluetooth headphones, speakers, HDMI — and audio plays on both simultaneously with proper sync.

It uses **native PipeWire graph linking** (`pw-link`) instead of PulseAudio compatibility modules, which means it actually works reliably and keeps devices in the running state without suspending.

When you quit, everything is restored exactly to how it was before.

---

## Features

- **Any device combo** — AUX + Bluetooth, AUX + AUX, BT + BT, USB + anything, HDMI + anything
- **Native PipeWire routing** via `pw-link` — no PulseAudio compat hacks, no sink suspension issues
- **BT + BT mode** — enables BlueZ MultiProfile for two simultaneous Bluetooth A2DP connections on one adapter
- **Auto-scan** — waits and detects devices automatically if only one is connected at launch
- **Smart device picker** — shows device type (AUX / Bluetooth / USB / HDMI) with icons, excludes already-chosen device from second pick
- **Duplicate cleanup** — detects and purges leftover modules from any previous crashed run before starting
- **Stream watcher** — background process re-routes any streams WirePlumber moves off the hub
- **Full restore on exit** — removes all virtual sinks and links, restores your previous default output, whether you press `q`, Ctrl-C, or the process is killed
- **Live controls** — volume up/down, mute/unmute while running

---

## Supported device combinations

| Device 1 | Device 2 | Works? | Notes |
|----------|----------|--------|-------|
| AUX | AUX | ✅ | Second AUX = USB DAC or separate card |
| AUX | Bluetooth | ✅ | Most common combo, works great |
| AUX | USB Audio | ✅ | |
| AUX | HDMI | ✅ | |
| Bluetooth | Bluetooth | ⚠️ | Depends on adapter hardware — see BT+BT section |
| USB | Bluetooth | ✅ | |
| HDMI | Bluetooth | ✅ | |

---

## Requirements

| Package | Purpose |
|---------|---------|
| `pipewire` | Core audio server |
| `pipewire-pulse` | PulseAudio compatibility layer |
| `wireplumber` | PipeWire session manager |
| `bluez` | Bluetooth stack |
| `bluez-utils` | `bluetoothctl` command |

Install everything:

```bash
sudo pacman -S pipewire pipewire-pulse pipewire-alsa wireplumber bluez bluez-utils
systemctl --user enable --now pipewire pipewire-pulse wireplumber
sudo systemctl enable --now bluetooth
```

---

## Installation

```bash
git clone https://github.com/yourusername/bitaud.git
cd bitaud
sudo cp bitaud /usr/local/bin/bitaud
sudo chmod +x /usr/local/bin/bitaud
```

---

## Usage

```bash
bitaud
```

bitaud will:
1. Check all dependencies
2. Wait for two audio devices to be available (scans automatically if only one is connected)
3. Show a numbered list of available devices — pick primary, then secondary
4. Wire up the audio graph and start playing to both
5. Stay running with live controls until you quit

### Controls

| Key | Action |
|-----|--------|
| `+` or `=` | Volume up 5% |
| `-` | Volume down 5% |
| `m` | Mute / unmute |
| `q` | Quit and restore audio |
| `Ctrl-C` | Quit and restore audio |

---

## How it works

```
┌─────────────────────────────────────────────────────┐
│                    Your Apps                        │
│          (browser, Spotify, mpv, etc.)              │
└──────────────────────┬──────────────────────────────┘
                       │  audio stream
                       ▼
┌─────────────────────────────────────────────────────┐
│              bitaud_hub  (null sink)                │
│         virtual default output device               │
└────────────┬────────────────────┬───────────────────┘
             │  pw-link           │  pw-link
             ▼                    ▼
┌────────────────────┐  ┌────────────────────────────┐
│   Device 1         │  │   Device 2                 │
│   AUX / BT / USB   │  │   AUX / BT / USB / HDMI    │
└────────────────────┘  └────────────────────────────┘
```

bitaud creates a **virtual null sink** (hub) and sets it as the system default. All audio from your apps flows into the hub. It then uses `pw-link` to create native PipeWire graph connections from the hub's monitor ports directly to the playback ports of both real output devices.

**Why `pw-link` instead of `module-loopback`?**

`module-loopback` is a PulseAudio module that PipeWire emulates for compatibility. It has a known issue where it doesn't prevent PipeWire from auto-suspending idle sinks, causing audio to stop after a few seconds. `pw-link` creates real PipeWire graph links at the node level — these keep the connected nodes in RUNNING state as long as the link exists.

---

## BT + BT mode

When you select two Bluetooth devices, bitaud automatically enters BT+BT mode:

1. Backs up `/etc/bluetooth/main.conf`
2. Enables `MultiProfile = multiple` and `Experimental = true` in BlueZ config
3. Restarts the Bluetooth service
4. Auto-reconnects both headphones via `bluetoothctl`
5. Waits for both BT sinks to appear in PipeWire
6. Creates the hub and pw-links as normal
7. Fully restores BlueZ config on exit

> **Note:** Success depends on your Bluetooth adapter hardware. Some adapters support two simultaneous A2DP connections, others don't at the firmware level. bitaud will tell you clearly if it doesn't work and fall back gracefully. Your hardware cannot be damaged by trying.

---

## Troubleshooting

**PipeWire not running**
```bash
systemctl --user status pipewire pipewire-pulse wireplumber
systemctl --user start pipewire pipewire-pulse wireplumber
```

**Leftover modules from a crashed run**
```bash
pactl list short modules | grep -E "null|loopback" | awk '{print $1}' | xargs -I{} pactl unload-module {}
```

**Check connected devices**
```bash
pactl list short sinks
pw-link -i
```

**Bluetooth device not showing up**
```bash
bluetoothctl
power on
connect XX:XX:XX:XX:XX:XX
```

---

## Why I built this

Every existing solution for dual audio on Linux either used deprecated PulseAudio modules that don't work properly under PipeWire, required a GUI, or silently broke due to sink auto-suspend. bitaud is a proper PipeWire-native CLI tool that actually works.

---

## Contributing

Issues and PRs welcome. If you find a device combo that doesn't work, open an issue with the output of:

```bash
pactl list short sinks
pw-link -i
wpctl status
```

---

## License

MIT — see [LICENSE](LICENSE)
