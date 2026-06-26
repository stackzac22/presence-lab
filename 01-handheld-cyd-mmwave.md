# 01 — Handheld CYD mmWave detector

**Goal:** a pocket "is there a human here (even still, even in the dark)" device — CYD
screen shows presence + distance live, runs off a power bank. Also doubles as a
**site-survey tool** to scout where to place fixed nodes later.

## Bill of materials
- 1× **CYD** (ESP32-2432S028R) — you have several
- 1× **LD2410** (or LD2410C/B) mmWave module
- Jumper wires (JST-to-Dupont for the CYD's side connectors helps)
- Power: USB power bank (simplest) — or vape cell + 3.7→5 V boost (see build 03)
- Optional: small 3D-printed/plastic shell

## Wiring (LD2410 → CYD)
The CYD breaks out a few GPIO on its side JST connectors (CN1/P3). **Verify against
your board's silkscreen** — pin labels vary by revision. Commonly free: **GPIO22,
GPIO27, GPIO35 (input-only)**, plus **5V/VIN** and **GND** on a connector.

Two options:

**A) Simple presence (easiest) — use the LD2410 OUT pin**
| LD2410 | → CYD | Note |
|---|---|---|
| 5V  | 5V/VIN | from a side connector |
| GND | GND | |
| OUT | GPIO35 | digital presence high/low (GPIO35 is input-only — perfect) |

→ Screen shows a big PRESENT / CLEAR. No distance, but dead simple.

**B) Full data — UART (distance, moving/still)**
| LD2410 | → CYD | Note |
|---|---|---|
| 5V  | 5V/VIN | |
| GND | GND | |
| TX  | GPIO27 | module TX → CYD RX |
| RX  | GPIO22 | module RX → CYD TX |

→ Screen shows distance + moving/still energy. Baud **256000**.

> ⚠️ The display + touch already use most CYD pins; the side-connector GPIOs above are
> the safe ones. Don't borrow display/SD pins.

## Firmware
- **ESPHome** is cleanest (you have it): CYD display platform + `ld2410` component +
  a `display:` lambda that draws presence/distance. Works standalone (no HA needed) —
  the screen renders regardless. Reflash the CYD from GhostESP → this ESPHome config.
- Alternative: an Arduino/LVGL sketch if you want fancier UI.
- (I can write the ESPHome YAML when you pick option A or B.)

## Power
- **Power bank** = no-fuss handheld. CYD + backlight + LD2410 ≈ 150–250 mA.
- Vape cell + boost works but expect only a few hours (see 03). Dim the backlight to stretch it.

## What "good" looks like / test
- Detects you standing **still** at 4–6 m (breathing micro-motion).
- Detects through a closed **interior door**? (note pass/fail)
- Walk it room-to-room as a **survey tool**: where does it reliably see vs miss → that's your fixed-node map.

## Limits
- ~6 m, ~120° cone. Through glass/thin drywall only — not brick/concrete.
- Can't tell human from large pet.

## Add-to-plan ideas
- Add a **buzzer** (CYD speaker on GPIO26) for an audible chirp on detect — hands-free sweeping.
- Log distance to the CYD's **SD card** while surveying → import the heatmap later.
- This handheld is the perfect tool to **place the build-02 outdoor nodes** and the future mesh.
