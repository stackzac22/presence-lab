# 02 — Low-voltage outdoor node (12 V run)

**Goal:** a fixed, always-on outdoor node with **zero battery anxiety** — powered by a
12 V DC wire run, weatherproofed, reporting wirelessly to HA. Primary role: a **CSI
mesh node** (through-wall/yard motion); can alternatively host an LD2410.

## Why 12 V down the wire (not 5 V)
Long thin wire = voltage drop. 5 V can't spare any; **12 V has headroom**, so you run
12 V out and step down **at the node** with a tiny buck. Same idea as landscape lighting:
safe, no permits, staple it along the fence.

## Bill of materials
- 1× **ESP32** (classic; **external-antenna / u.FL version** strongly preferred outdoors)
- 1× **12 V DC supply** (1 A wall brick is plenty) at the indoor/garage end
- **18–22 AWG 2-conductor wire**, length to reach (50–100 ft fine at this current)
- 1× **MP1584 mini buck** (12 V → 5 V), ~$1
- 1× **IP65 enclosure** + cable gland + **desiccant pack**
- Rigid mount (post / fence rail) — **NOT** inside a swaying bush
- (CSI mesh needs a partner: a TX node running `csi_send`)

## Power chain
`12 V brick (indoors) → 2-wire run → MP1584 (set to 5.0 V) → ESP32 5V/VIN + GND`
- Node draws ~0.15 A → wire current is tiny, drop is negligible.
- Set the MP1584 output to **5.0 V with a multimeter before connecting the ESP.**

## Firmware — the architecture change (two paths)
Indoors your CSI RX (`/home/tec/esp/csi_radar`, esp-csi `console_test`, **IDF 5.4** — v6
breaks it) is **USB-tethered** to a host running `csi_ha_bridge.py`. Outdoors there's no
host. Two ways to solve it:

### Path 2a — Pi-in-the-box (FASTEST, zero firmware change) ✅ start here
Put a **Pi Zero 2 W** (~$15) in the weatherproof box next to the ESP32. ESP32 stays
**unmodified** (console_test over USB → local Pi); the Pi runs your **existing
`csi_ha_bridge.py`** and publishes to HA over WiFi. The **12 V run powers both** (buck →
5 V → Pi + ESP).
- Pros: build & test the *whole* outdoor concept (power, weather, detection, HA reporting) **today**, no C work.
- Cons: a Pi per node; a bit more power.

### Path 2b — onboard MQTT (CLEANER, real firmware work)
Modify the `console_test` copy so the ESP32 publishes `someone`/`move` itself:
1. Add **WiFi STA** (join home AP) + **esp-mqtt** client to the project.
2. In the radar prediction path (where it builds the `RADAR_DADA` line in
   `main/app_main.c`), add `esp_mqtt_client_publish(...)` of the someone/move flags.
3. HA gets it via MQTT discovery, same as your current entities.
- ⚠️ **The one real gotcha — channel coexistence:** CSI capture and the STA link must be on
  the **same WiFi channel**. So set the **TX (`csi_send`) to the home AP's channel**; the RX
  joins the AP (→ that channel) and captures the TX's CSI on it. If they're on different
  channels, you get MQTT *or* CSI, not both.
- Pros: no Pi, lowest power/parts. Cons: C firmware + iterate.

### Either way (or skip CSI here)
- **Alt sensor:** run **ESPHome + LD2410** on this node instead — clean ~6 m presence,
  trivial config, if you don't need through-wall *at this spot*.
- **Recommendation:** build the box on **Path 2a** first (prove power/weather/detection),
  then graduate the *same hardware* to **2b** once it's earning its keep.

## Weather + placement (the part that bites)
- IP65 box, gland the wire, desiccant inside (condensation kills these).
- **Mount rigid** — wind-blown foliage disturbs the CSI links → false trips. Camouflage the
  box on a post; don't bury it *in* the bush.
- External antenna out the bottom of the box (water shedding).

## What "good" looks like / test
- Detection range across the yard along the **link path** to the indoor anchor.
- **False-positive rate over 1 hr idle** (wind/rain/cats) — the key outdoor metric.
- Link stays associated (no dropouts) at the install distance.
- Survives a rain + a hot afternoon (check for condensation).

## Limits
- CSI = "movement somewhere along the link," not position or species.
- Coverage = the link lines; 1–2 nodes cover main approach lanes, not a guaranteed whole yard.

## Add-to-plan ideas
- **Fuse with the XIAO cam + Frigate**: CSI says "movement," camera confirms "*person*" → kills animal/wind false alarms.
- Run **two outdoor nodes** so their links cross → rough *which-corner* localization.
- Later: make the indoor **ESP32-C5** the dual-band hub for sharper 5 GHz CSI.
