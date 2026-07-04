<div align="center">

# 📡 SignalHunter
### *by Patch*

<img width="600" height="400" alt="image" src="https://github.com/user-attachments/assets/9f828b6c-ff85-424c-9e1b-3dd7d4c87743" />


**A pocket-sized BLE & WiFi field scanner for the ESP32 "Cheap Yellow Display"**

![Platform](https://img.shields.io/badge/platform-ESP32-orange?style=flat-square)
![Framework](https://img.shields.io/badge/framework-Arduino%20%2F%20PlatformIO-00979D?style=flat-square)
![Language](https://img.shields.io/badge/language-C%2B%2B-blue?style=flat-square)
![Status](https://img.shields.io/badge/status-v1.0%20stable-brightgreen?style=flat-square)
![Scope](https://img.shields.io/badge/scope-passive%20%2F%20receive--only-lightgrey?style=flat-square)

*A $15 dev board that fits in your pocket, tells you what's in the air around you,*
*and doesn't need a laptop, a Pi, or root access to do it.*

</div>

---

## What is this thing

SignalHunter turns a cheap ESP32 "CYD" (Cheap Yellow Display) board into a
standalone, no-laptop-needed radio awareness tool. Flip it on, and it
continuously scans the BLE and WiFi environment around you, showing what's
nearby, how strong it is, and flagging the device types most people actually
care about spotting: **trackers, skimmers, and surveillance gear.**

It's the little sibling of [BTScanner](../btscanner/), a full Python/Textual
BLE threat-detection dashboard for Linux/Pi — SignalHunter is the field
version: no laptop, no database, just a screen and a battery.

```
┌─────────────────────────────────────────────────────────────────┐
│ [≡] SignalHunter          🟢          N:12          0:04:22      │
│      Skimmer Finder                                              │
├── ALL ── SKIM ──[TAG]── FLIP ── AXON ── WIFI ─────────────────── │
│ ▌ unnamed a1b2c        [TAG]        ▂▄▆█  -58                     │
│ ▌ HC-05                [SKIM]       ▂▄▆_  -71                     │
│ ▌ Jay's iPhone          Apple       ▂▄▆█  -49                     │
│ ▌ unnamed e5f9c                     ▂▄__  -82                     │
│ ▌ Flipper 3a7f          [FLIP]      ▂▄▆_  -63                     │
├────────────────────────────────────────────────────────────────── │
│              tap = next mode, hold = pause                       │
└─────────────────────────────────────────────────────────────────┘
```
*(what the screen actually looks like — real photos welcome via PR!)*

---

## ✨ Features

- **Six live modes**, cycled with a tap or the BOOT button:
  | Mode | What it shows |
  |------|----------------|
  | **ALL** | Every BLE device in range, sorted strongest-signal-first |
  | **SKIM** | 🔴 Card-skimmer Bluetooth modules (HC-05/06/08/03, FREE2MOVE, etc.) |
  | **TAG** | 🔴 AirTags & FindMy-network trackers (UUID + Apple mfr-data match) |
  | **FLIP** | 🟡 Flipper Zero devices |
  | **AXON** | 🟡 Body cameras / tasers (Axon, evidence.com) |
  | **WIFI** | 📶 Nearby access points — SSID, channel, encryption, signal |
- **Continuous background scanning** — the radio never blocks the screen;
  navigation stays snappy no matter what's happening on air.
- **One-handed control** — short tap/press = next mode, long-press
  (~600ms) = pause/resume scanning.
- **Color-coded everything** — signal-strength bars, threat badges, and a
  live scanning indicator, all readable at a glance.
- **Fully passive.** SignalHunter only *listens*. It never transmits deauth
  frames, spam advertisements, or anything else aimed at someone else's
  device — see [Scope & Ethics](#-scope--ethics) below.

---

## 🛠 Hardware

Built for the **ESP32-2432S028R** — the most commonly sold "CYD": a 2.8"
ILI9341 320×240 display with resistive touch, on an ESP32-WROOM-32.

> **Different CYD variant?** (capacitive/GT911 touch, 3.5" panel, different
> silkscreen) — the pin map below may not match. Flip the board over; the
> silkscreen usually prints the exact model. See
> [Troubleshooting](#-troubleshooting) if the screen looks wrong on first boot.

Display/touch pins are set entirely via `build_flags` in
[`platformio.ini`](platformio.ini) — no hand-editing library files:

| Signal | Pin | Signal | Pin |
|--------|-----|--------|-----|
| MISO   | 12  | DC       | 2   |
| MOSI   | 13  | RST      | -1 (tied to EN) |
| SCLK   | 14  | BL       | 21  |
| CS     | 15  | TOUCH_CS | 33  |

---

## 🚀 Quick start

### Option A — flash a prebuilt release (fastest)

Grab the latest `.bin` from [`releases/`](releases/) and flash it directly:

```powershell
pip install esptool
esptool.py --chip esp32 --port COM8 --baud 921600 write_flash 0x10000 "releases/Patch Signal Hunter v1.0.bin"
```

(Swap `COM8` for your board's port — see [Finding your port](#finding-your-port) below.)

### Option B — build from source

```powershell
pip install platformio
cd cyd_scanner

pio run                              # compile only — verify before touching hardware
pio device list                      # find your board's COM port
pio run --target upload --upload-port COM8
pio device monitor                   # watch live serial output (Ctrl+C to exit)
```

Most CYD boards auto-enter bootloader mode on upload via the USB-serial
chip's DTR/RTS lines — no BOOT-button-holding needed. If upload fails or
times out, hold **BOOT**, unplug/replug the USB cable while still holding
it, wait ~1s, release, then retry the upload — this forces bootloader mode
regardless of the auto-reset circuit.

#### Finding your port

```powershell
pio device list
```
Look for `USB-SERIAL CH340` or `CP210x` — that's your board.

---

## 🎮 Controls

| Input | Action |
|-------|--------|
| **Tap screen** (short) | Next mode |
| **BOOT button** (short press) | Next mode (works even if touch is acting up) |
| **Hold ~600ms** (touch or BOOT) | Pause / resume scanning |

---

## 🔍 How the Finder modes work

The four Finder modes are a C++ port of the classification logic from the
[Python BTScanner's `classifier.py`](../btscanner/classifier.py) — same
detection layers, trimmed down for a 320KB-RAM chip:

1. **Name-pattern match** — e.g. `HC-05`/`FREE2MOVE` → Skimmer, `flipper` → Flipper Zero
2. **Service UUID match** — AirTag/FindMy UUIDs, Flipper Zero's advertised service
3. **Apple manufacturer-data refinement** — distinguishes an AirTag from a
   generic Apple device by its Continuity message byte pattern

WiFi Recon is the same spirit applied to WiFi: nearby access points, signal,
channel, and encryption type — nothing transmitted beyond a standard scan
(the same one your phone runs to list nearby networks).

---

## 🧠 Architecture notes (for contributors)

- **BLE scanning runs continuously** in NimBLE's own background task
  (`duration=0`) — the main loop never blocks waiting on it, which is what
  keeps the UI responsive regardless of scan activity.
- Every ~20s the scan does a brief **stop/restart** to reset NimBLE's
  internal results cache — left unbounded, that cache grows for as long as
  the scan runs and can degrade performance (or crash) on a 320KB chip.
  ⚠️ Calling `clearResults()` directly on an *active* scan crashes the BT
  controller (a core-pinning assertion) — always go through `stop()`/`start()`.
- **WiFi scanning** is inherently a channel-hopping burst, so it's driven as
  a small non-blocking state machine (`startAsync` → `poll` → `collect`)
  spread across loop iterations instead of one blocking call.
- **Touch/button input is edge-detected** (fires once per tap/press, not
  once per poll while held) and classified as short-vs-long by hold duration.
- Display config lives entirely in `platformio.ini` `build_flags` — see
  [`theme.h`](src/theme.h) / [`mascot.h`](src/mascot.h) for the UI palette
  and logo, [`ble_scanner.*`](src/ble_scanner.cpp) and
  [`wifi_scanner.*`](src/wifi_scanner.cpp) for the scan logic, and
  [`ui.*`](src/ui.cpp) for rendering.

---

## 📦 Releases

Every validated release lives in [`releases/`](releases/) as a named,
never-overwritten `.bin` with a SHA256 sidecar — see
[`releases/CHANGELOG.md`](releases/CHANGELOG.md) for what changed in each
version and the exact rollback command.

---

## 🩹 Troubleshooting

<details>
<summary><strong>Blank / white / garbled screen</strong></summary>

Almost always a TFT_eSPI driver mismatch between clone panels. This repo
ships with `ILI9341_2_DRIVER` (not the plain `ILI9341_DRIVER`) because the
common CYD panel needs the alternate init sequence — if yours still looks
wrong, try flipping that line in `platformio.ini` and reflashing.
</details>

<details>
<summary><strong>Screen shows static/noise near an edge</strong></summary>

This is an SPI clock signal-integrity issue on cheap panel cables — this
repo runs at 27MHz (`SPI_FREQUENCY`) instead of the more common 40MHz for
exactly this reason. If you still see it, try dropping further (20MHz).
</details>

<details>
<summary><strong>Screen stays black, backlight is on</strong></summary>

Check `TFT_BL` polarity — some boards need the backlight pin driven LOW
instead of HIGH. Try flipping `TFT_BACKLIGHT_ON` in `platformio.ini`.
</details>

<details>
<summary><strong>No devices ever appear</strong></summary>

Confirm something nearby is actually advertising (a phone with Bluetooth
on, headphones, etc.), and check the serial monitor (`pio device monitor`)
for NimBLE init errors.
</details>

<details>
<summary><strong>Upload fails / times out</strong></summary>

Hold **BOOT**, unplug/replug USB while holding it, wait ~1s, release, then
retry — forces bootloader mode without relying on the auto-reset circuit.
Also try a lower `upload_speed` in `platformio.ini` (e.g. `460800`) if the
cable/hub is flaky.
</details>

<details>
<summary><strong>COM port shows "Unknown" status / won't open (Windows)</strong></summary>

A CH340 driver hiccup — try a different physical USB port first, then a
different cable. If that fails, open Device Manager, find the CH340 device
under Ports, and Disable → Enable it (or Uninstall device, then replug so
Windows reinstalls the driver fresh).
</details>

---

## 🗺 Roadmap

- **Direct tab-tap navigation** — jump straight to a mode by tapping its
  tab, once touch calibration is dialed in (right now any tap just advances
  to the next mode, which sidesteps needing calibration at all).
- **MQTT uplink** — report sightings to a central BTScanner instance for
  fingerprinting, persistence scoring, and multi-node correlation.
- **Threat coloring pushed from the central brain** once multi-node sync
  exists, so field units get the full scoring picture, not just raw RSSI.
- **Battery/portable enclosure** guide for real field use.

---

## 🎯 Scope & Ethics

SignalHunter is a **passive, receive-only** tool, on purpose. It listens to
what's already being broadcast — it never connects to, spoofs, floods, or
disrupts anyone else's device or network. There's no deauther, no beacon
spam, no BLE spam, no probe-flood, by design — those are attack tools aimed
at other people's devices, not scanning, and they're a different category of
thing than what this project is for.

Use it to understand the RF environment around **you** — not to track,
disrupt, or surveil anyone else. Know your local laws around radio
monitoring before you go hunting for signals in public.

---

## 🙏 Credits

Built by **Patch**. 

*This came from my other project BTScanner a Python bluetooth tracking system
displayed via Textual for Linux & Raspberry Pi*

## 📄 License

*Not yet chosen — add a `LICENSE` file before relying on this being open
source in any particular way.*
