# 04 — C5 + mmWave USB dongle

**Goal:** plug the **Waveshare ESP32-C5-DevKitC-1 + HLK-LD2420** into a host (Pi /
laptop / Android) over USB and stream clean presence data — a pocket "is-a-human-here"
**dongle** that any host can read.

> ⚠️ This board currently runs **`board-activity-01`** (the cornhole LD2420 node). Flashing
> the dongle firmware **removes it from the cornhole system.** Use a spare C5 or accept the reassignment.

## Hardware (already wired)
- **ESP32-C5-DevKitC-1** (`board: esp32-c5-devkitc-1`, `variant: esp32c5`)
- **HLK-LD2420** mmWave on **UART1**: LD2420 TX→**GPIO4** (ESP RX), LD2420 RX→**GPIO5** (ESP TX), 3V3, GND
- **USB:** CH343 on **UART0 (GPIO11/12)** → host sees **`/dev/ttyACM0`** (Linux/Pi), a COM port (Windows), or USB-OTG serial (Android)
- LD2420 talks **115200** (≠ LD2410's 256000 — different chip/protocol)

## Firmware — two modes

### D1 — Smart JSON dongle (recommended)
ESP parses the LD2420 and prints **clean JSON lines** over USB, so the host does nothing but read lines:
```
{"present":true,"moving":true,"still":false,"distance_cm":143}
```
Cleanest path reuses ESPHome (you already have `ld2420` working on this board):
- `logger: { baud_rate: 0 }`  → free UART0 from log spam (logs still available over WiFi if enabled)
- add a `uart` on **GPIO11(tx)/GPIO12(rx) @115200** = the CH343/USB link, id `usb_out`
- keep the `ld2420` on UART1 (GPIO4/5) as today
- an `interval: 250ms` lambda builds the JSON from the ld2420 sensors and `id(usb_out).write_str(...)`
- Bonus: it can **still** join WiFi/MQTT too → USB dongle *and* HA node at once.

### D2 — Transparent bridge (fallback)
Firmware just copies **UART1 ↔ UART0**, so the host talks the **raw LD2420 protocol** and parses it itself. Almost no firmware, but the host needs an LD2420 parser (you've got serial-parsing code already). Less friendly than D1.

## Host side
- **Pi / laptop:** read `/dev/ttyACM0` @115200 — `cat`, or a tiny Python (`pyserial`) loop. Trivial with D1's JSON.
- **Android:** USB-OTG + a serial terminal app (e.g. *Serial USB Terminal*) shows the JSON; a custom app can parse it. **This is the "plug into phone" path.**
- **CYD:** ❌ a CYD is a USB *device*, not a host — it can't read the dongle over USB. To pair with a CYD, use **ESP-NOW/WiFi** instead, not USB.
- **iPhone:** ❌ no third-party USB serial.

## Power
USB-powered by the host (or a power bank). C5 + LD2420 draw is modest — fine off a port.

## What "good" looks like / test
- Clean JSON at 250 ms on `/dev/ttyACM0`, presence flips within ~0.5 s of you entering.
- Works the same plugged into Pi **and** Android (OTG) — that's the "mobile dongle" win.
- Note detection range (LD2420 ~ up to ~8 m configurable) + still-target behavior.

## Add-to-plan ideas
- D1 dongle doubles as a **wireless HA node** (keep WiFi/MQTT on) — USB for mobile, MQTT when parked.
- Tiny Android app later: parse the JSON → vibrate/alert = a true handheld "bug/human sweep."
- Pair-mode over **ESP-NOW** to talk to a CYD screen (since USB-to-CYD won't work).

---

## Build log / status — 2026-06-26 ✅ FLASHED & VERIFIED (D1 mode)

Firmware **D1 (Smart JSON dongle)** is live on the board. Config: `/home/tec/HA/c5-mmwave-dongle.yaml`.

**Flash command** (esphome is in the venv, not on PATH):
```bash
cd /home/tec/HA && .esphome-venv/bin/esphome run c5-mmwave-dongle.yaml --device /dev/ttyACM0
```

**Verified working:**
- USB JSON streaming on `/dev/ttyACM0` @115200 — `{"present":true,"distance_cm":105}` (set baud first: `stty -F /dev/ttyACM0 115200 raw -echo`). Radar detected presence at ~105 cm.
- WiFi + MQTT up: board published HA discovery → `homeassistant/binary_sensor/c5-mmwave-dongle/presence/config`. Broker `<your-broker-ip>`. Shows in HA as **"C5 mmWave Dongle"** (Presence/occupancy, Distance, WiFi RSSI). So it's a USB dongle **and** a wireless HA node at once, as planned.

**Gotcha fixed:** `secrets.yaml` runs values through ESPHome substitution, so a literal `$` must be `$$`. An SSID containing a literal `$` (e.g. `My$Net`) makes ESPHome log a cosmetic warning (`looks like an expression … is undefined`) because `$` reads like a substitution. But ESPHome keeps the value **literal and connects fine** — leave the `$` as-is. Do **not** "escape" it as `$$`; that sends a literal double-`$` and actually breaks WiFi.

**Recovery note:** an interrupted flash (wrong button) does **not** brick the C5 — it falls back to ROM download mode. Verify with `esptool --port /dev/ttyACM0 chip-id`, then just re-run the flash command above.

**Still TODO:** measure detection range (move/still), latency, idle false-positive rate, current draw for the bake-off table; test the Android USB-OTG path.
