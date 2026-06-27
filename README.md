# 🛰️ Presence Lab — RF human-sensing test bench

Three build variants to prototype and **bake-off** against each other, then fold the
winners into the bigger plan (whole-home CSI mesh + yard perimeter + camera fusion).

| # | Build | Sensor | Power | Role it's testing |
|---|-------|--------|-------|-------------------|
| [01](01-handheld-cyd-mmwave.md) | **Handheld CYD** | LD2410 mmWave | USB / power bank (or vape+boost) | portable "is a human here" with a screen |
| [02](02-lowvolt-outdoor-csi-node.md) | **Low-volt outdoor node** | WiFi CSI (or LD2410) | 12 V wire run → buck | always-on fixed node, no battery anxiety |
| [03](03-vape-battery-node.md) | **Vape-battery node** | PIR or LD2410 | 3.7 V vape cell | how long a salvaged cell lasts; sleepy-sensor duty |
| [04](04-c5-mmwave-dongle.md) | **C5 + mmWave USB dongle** | LD2420 on C5 | host USB / power bank | plug into Pi/phone, stream presence JSON |
| [05](05-ld2450-multitarget.md) | **LD2450 multi-target** | LD2450 on C5 | host USB / power bank | track up to 3 people (X/Y), people-counting |

## Bake-off — record these for each build
Drop results in `results.md` (make it as you go):

- **Detection range** (m) — moving target, and *stationary* target (breathing micro-motion)
- **Latency** — motion → alert/screen (ms/s)
- **Power draw** — average mA (use a USB meter / multimeter)
- **Runtime** — on battery builds, hours per charge
- **False-positive rate** — over a 1 hr idle window (pets, wind, HVAC)
- **Through-material** — none / glass / drywall / brick
- **Survivability** — heat, damp (outdoor build)

## How these ladder into the big plan
1. **Handheld (01)** = the cheap fast win + a roving diagnostic tool (walk it around to scout where fixed nodes should go).
2. **Outdoor node (02)** = first yard node; proves the 12 V run + weatherproofing + wireless CSI reporting (the architecture change off USB-tether).
3. **Battery node (03)** = decides whether *any* battery sensor is viable, and which sensor (PIR sips, mmWave gulps).
4. Then scale: **10-node CSI mesh** (zone map in HA) + **C5 as dual-band hub** + **XIAO cam/Frigate** as the human-vs-animal confirmer + **LD2410 at the slider** as the close-range backstop.

## Sensor cheat-sheet (so the docs don't repeat it)
- **LD2410** — UART @ 256000 baud, OR a digital **OUT** pin (presence high/low). ~6 m range, sees moving + *still* (breathing) humans, works in the dark, through glass/thin drywall. Always-on (~80 mA radar).
- **WiFi CSI** — needs a **TX node + RX node**; senses disturbance to the link between them → through-wall *motion*. Radio always on (power hog). esp-csi firmware, **ESP-IDF 5.4** (v6 breaks it).
- **PIR** — passive, µA idle, wakes ESP on motion. Cheap, line-of-sight only, motion-only (no stationary), no through-wall. The battery-friendly option.
