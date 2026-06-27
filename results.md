# Presence Lab — bake-off results

| Build | Detect range (move/still) | Latency | Avg mA | Runtime | False+/hr idle | Through | Notes |
|-------|---------------------------|---------|--------|---------|----------------|---------|-------|
| 01 Handheld CYD |  |  |  | (power bank) |  |  |  |
| 02 Outdoor CSI  |  |  |  | (wired) |  |  |  |
| 03 Vape PIR     |  |  |  |  |  |  |  |
| 03 Vape LD2410  |  |  |  |  |  |  |  |
| 04 C5 mmWave dongle | **~831 / ~831 cm** (8.3 m move & still) | immediate (`present` by 1 s) | TBD | (USB/bank) | **0** (3×10min) | USB JSON + MQTT | ✅ flashed & verified 2026-06-26; USB `/dev/ttyACM0` + HA node both live |
| 05 LD2450 multi-target | TBD (~6 m, moving) | TBD | TBD | (USB/bank) | TBD | USB JSON + MQTT | 📋 spec'd — tracks up to 3 targets X/Y; ⚠️ radar UART @ **256000** baud |

## Observations
- **04 C5 dongle (2026-06-26):** D1 JSON firmware verified — clean `{"present":...,"distance_cm":...}` on `/dev/ttyACM0` and HA discovery published over MQTT. Fixed a `secrets.yaml` `$`-escaping bug (SSID needs `$$`) that had blocked WiFi.
- **04 idle false-positive cal (2026-06-26):** 3×10 min, nobody in front, read off the Nexus 5X over USB-OTG. **0 false positives** across ~7,170 samples (runs: 2401/2384/2385, all 0 `present:true`) → **0 False+/hr**. Clean.
- **04 range walk test (2026-06-26):** clean phone-timestamped 75 s capture, read off Nexus 5X over OTG. Tracked 100% continuously while walking out to **831 cm (8.3 m)**, then **held a stationary target at 831 cm for ~20 s** (still-hold range also ~8.3 m). Immediate latency (`present:true` within 1 s). Only flickered at the very edge (~8.3 m). Effectively full LD2420 range for both moving and still. NOTE: needed an LD2420 wire reseat first — a loose radar wire makes the dongle stream `distance_cm:null`/no detection while the ESP keeps running.
- Battery (rough): 79%→48% over ~7h16m with apps off = ~4.3%/hr, but the span wasn't verified true-idle — treat as loose, not the official Avg mA figure.
- 

## Decisions
- 
