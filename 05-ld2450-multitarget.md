# 05 — LD2450 multi-target tracker (up to 3 people, X/Y)

**Goal:** step up from single-presence (build 04's LD2420) to a **multi-target** radar that
**tracks up to 3 moving people at once**, each with **position (X/Y), speed, and distance** —
so we can actually *count* and *locate*, not just say "someone's here." Same C5 dongle idea:
stream rich JSON over USB **and** publish per-target entities to Home Assistant.

> This is the answer to "can we calibrate for more than one person?" — the LD2420 can't
> (one boolean + one distance). The **HLK-LD2450** can.

## What changes vs build 04 (LD2420)
| | LD2420 (build 04) | **LD2450 (this build)** |
|---|---|---|
| Targets | 1 (presence only) | **up to 3, tracked simultaneously** |
| Per-target data | distance only | **X, Y, speed, distance, angle** |
| Counting people | ❌ | ✅ (0–3) |
| Localization | ❌ | ✅ (X/Y in mm, ±60° FoV) |
| **UART baud** | **115200** | **256000** ⚠️ (different — don't reuse 04's baud) |
| Best at | still presence (breathing) | **moving** targets (Doppler tracker; weaker on dead-still) |
| Range | ~8 m | ~6 m |

⚠️ **Biggest gotcha: the LD2450 talks at 256000 baud, not 115200.** Carrying over build 04's
`baud_rate: 115200` on the radar UART = garbage/no data. (Same trap as LD2410, also 256000.)

## Hardware
- **ESP32-C5-DevKitC-1** (reuse build 04's board, or a spare so 04 stays intact)
- **HLK-LD2450** 24 GHz radar, FoV **±60°**, range **~6 m**, tracks **3** moving targets
- Wiring (same crossover idea as 04, **but radar UART @ 256000**):

| LD2450 | → ESP32-C5 | note |
|--------|-----------|------|
| **TX** | GPIO4 | ESP RX (radar → ESP) |
| **RX** | GPIO5 | ESP TX (ESP → radar) |
| **3V3** | 3V3 | power (3.3 V, not 5 V) |
| **GND** | GND | ground |

- USB/JSON out stays on the CH343 / `usb_out` UART (GPIO11/12 @115200) → host `/dev/ttyACM0`.

## Where to buy
- **Amazon** (fast, ~$15–25): <https://www.amazon.com/s?k=HLK-LD2450>
- **AliExpress — Hi-Link Official Store** (cheapest, ~$6–12, slow ship): <https://www.aliexpress.com/wholesale?SearchText=HLK-LD2450>

**What to order:**
- The module labeled **HLK-LD2450** (24 GHz, multi-target) — *not* the LD2410 (single-presence) or LD2420 (build 04).
- Ships as a bare board with a **4-pin connector**; make sure a **JST pigtail cable** is included (or grab one) for TX / RX / 3V3 / GND leads.
- On AliExpress, buy from the **Hi-Link Official Store** for the genuine part.
- A version with a USB-C dev adapter exists — handy but unnecessary here, since it wires to the C5.

## Firmware — ESPHome `ld2450` component
ESPHome has a native `ld2450:` platform. Reuse build 04's structure, swap the radar block:

```yaml
uart:
  - id: radar_uart          # LD2450 — NOTE 256000 baud
    tx_pin: GPIO5
    rx_pin: GPIO4
    baud_rate: 256000
    parity: NONE
    stop_bits: 1
  - id: usb_out             # CH343 -> /dev/ttyACM0 (unchanged from build 04)
    tx_pin: GPIO11
    rx_pin: GPIO12
    baud_rate: 115200

ld2450:
  id: radar
  uart_id: radar_uart
  throttle: 250ms

# Per-target entities (HA auto-discovers via MQTT, same as 04)
sensor:
  - platform: ld2450
    ld2450_id: radar
    target_count: { name: "Target Count" }      # 0..3  <- the people counter
    target_1: { x: {name: "T1 X"}, y: {name: "T1 Y"}, speed: {name: "T1 Speed"}, distance: {name: "T1 Dist"} }
    target_2: { x: {name: "T2 X"}, y: {name: "T2 Y"}, speed: {name: "T2 Speed"}, distance: {name: "T2 Dist"} }
    target_3: { x: {name: "T3 X"}, y: {name: "T3 Y"}, speed: {name: "T3 Speed"}, distance: {name: "T3 Dist"} }
binary_sensor:
  - platform: ld2450
    ld2450_id: radar
    has_target: { name: "Presence" }
    has_moving_target: { name: "Moving" }
    has_still_target: { name: "Still" }
```

### D1 — multi-target JSON dongle (USB)
Same 250 ms `interval:` lambda as build 04, but emit an array — one object per active target:
```json
{"count":2,"targets":[
  {"id":1,"x":-312,"y":1480,"speed":-18,"dist":1513},
  {"id":2,"x":205,"y":2240,"speed":0,"dist":2249}]}
```
(x/y/dist in **mm**, speed in **cm/s**, negative speed = approaching.) Host reads `count` to
get the headcount and `targets[]` for positions — no parsing of raw radar protocol needed.

## Host side
- **Pi / laptop / Android-OTG:** identical to build 04 — read `/dev/ttyACM0` @115200, but now you
  get a people **count + coordinates** per line instead of one boolean.
- **HA:** `target_count` entity = live occupancy number; X/Y enable zone logic (e.g. "someone in
  the doorway zone") via ESPHome `zones:` or HA templates.

## What "good" looks like / test (the multi-person cal we couldn't do on 04)
- **Count accuracy:** 1, 2, then 3 people standing apart → `count` reads 1/2/3. Note where it
  miscounts (people too close together merge into one track).
- **Track separation:** two people crossing paths — do T1/T2 stay distinct or swap/drop?
- **Position sanity:** stand at known spots, check X/Y mm vs tape measure; map the ±60° edges.
- **Moving vs still:** confirm the known weakness — does a dead-still person drop off? (LD2450 is
  a motion tracker; budget for still-target loss vs the LD2420's strong still-hold.)
- **Range:** walk out to ~6 m; compare effective range to 04's 8.3 m.
- Re-use the same phone-timestamped capture script from build 04's range test.

## Power
USB-powered like 04; 24 GHz radar draw is modest. Log Avg mA with the USB meter for the bake-off.

## Add-to-plan ideas
- **Zone occupancy** (ESPHome `zones:`) → per-region people counts = the foundation for the
  whole-home zone map in HA, without the CSI mesh.
- LD2450 (position) + LD2420 (strong still-hold) on two boards = **count *and* don't-lose-the-
  napping-human** — complementary, fuse in HA.
- XIAO cam as the human-vs-pet confirmer on top of the count (ties into the camera-fusion plan).
