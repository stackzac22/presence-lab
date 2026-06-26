# 03 — Vape-battery node (runtime test)

**Goal:** find out how usable salvaged **3.7 V vape Li-ion cells** are for a wireless
sensor node — measure real runtime, and learn which sensor/duty-cycle a battery node
can actually sustain. (Spoiler from the math: continuous CSI = hours; a *sleepy* PIR =
weeks. This build proves it on your bench.)

## Bill of materials
- 1× salvaged **vape Li-ion cell** (3.7 V nom, has on/off switch + USB-C recharge — nice)
- 1× **TP4056** charge module *(if the cell doesn't already manage its own charging)*
- 1× **3.7 V → 5 V boost** module (e.g. MT3608) — OR an ESP board with native LiPo input
- 1× ESP32 (or cheaper **ESP32-C3** to sip less) 
- Sensor: **PIR** (battery-friendly) **and/or** **LD2410** (for the gulpy comparison)
- USB power/current meter (to log draw + capacity)

## ⚠️ Power wiring (don't fry the ESP)
A Li-ion is 4.2 V full → ~3.3 V empty. **Do NOT feed it straight to the 3.3 V pin.**
- `cell (3.7 V) → boost to 5.0 V → ESP 5V/VIN` (regulator on the board makes 3.3 V), or
- use a board with a proper LiPo connector + onboard charge/regulation.
- Vape cells are unprotected/cheap — use a holder, **don't over-discharge below ~3.0 V**,
  add a TP4056 (has protection) for charging. Respect them: they're real Li-ion (fire risk if abused).

## The runtime reality (why duty-cycle is everything)
| Sensor + mode | Avg draw | Runtime on ~400 mAh cell |
|---|---|---|
| **CSI, no sleep** | ~150–250 mA | **~2–3 hr** ❌ |
| **LD2410, always-on** | ~100–130 mA | ~3–4 hr |
| **LD2410, sample + light sleep** | ~20–40 mA | ~10–20 hr |
| **PIR + ESP deep-sleep (wake-on-motion)** | <1 mA idle, ms spikes | **days–weeks** ✅ |

→ For battery, the win is **PIR + deep sleep**: the ESP sleeps at µA, the PIR's motion
pin yanks it awake (`ext0` wake), it reports over WiFi/ESP-NOW, sleeps again.

## Firmware
- **ESPHome** with `deep_sleep:` + a PIR `binary_sensor` on a wake pin = the easy long-life build.
- LD2410 version: ESPHome `ld2410`, optionally `deep_sleep` with periodic wake (trades latency for life).
- (I can write either YAML.)

## What "good" looks like / test (the whole point)
Log to `results.md`:
- **mAh delivered** before cutoff (per cell — they vary a lot)
- **Runtime** for each mode above (CSI vs LD2410-always vs LD2410-sleep vs PIR-sleep)
- **Sleep current** measured (µA) — tells you the deep-sleep floor
- Recharge cycles before the cell noticeably fades

## Verdict it'll produce
- Continuous-RF on a vape cell = **bench/portable only** (hours).
- Sleepy PIR/door node = **genuinely deployable** for weeks → these are your "cheap battery tripwires."
- Pair with **solar** if you want an always-on sensor untethered (cell = night buffer).

## Add-to-plan ideas
- Build a few **PIR deep-sleep "tripwire" nodes** from vape cells for gate/side-yard — they
  feed the same HA presence map cheaply where running wire is a pain.
- Gang **2–3 vape cells in parallel** + a small **solar panel** → test an off-grid always-on node.
- Standardize on **ESP-NOW** for the battery nodes (lower power than full WiFi assoc).
