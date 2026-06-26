# C5 mmWave dongle — read it on Android (USB-OTG serial)

The dongle streams plain-text JSON over USB. Any Android phone with USB-OTG can read it — no app coding, no root.

## You need
- **USB-OTG adapter / cable** — USB-C (or micro-USB) phone → the dongle's USB cable.
- A **serial terminal app**. Recommended: **Serial USB Terminal** by Kai Morich (free, Play Store). Alternatives: *USB Serial Terminal*, *Termux* + `termios`.

## Steps
1. Plug the dongle into the phone via the OTG adapter. Android pops **"Allow app to access USB device?"** → tap **OK** (and tick "use by default" so it auto-connects next time).
2. Open **Serial USB Terminal**.
3. Set the connection (menu → **Settings → Connection**, or the gear):
   - **Baud rate: 115200**
   - **Data bits: 8 · Parity: None · Stop bits: 1**  (8N1)
   - DTR/RTS: leave default/off
4. Tap **Connect** (the plug/⚡ icon). The CH343 device should show as something like `CH343` / `USB Single Serial`.
5. You'll see lines streaming in:
   ```
   {"present":true,"distance_cm":105}
   {"present":false,"distance_cm":null}
   ```
   `present` flips true within ~0.5 s when someone's in front of it; `distance_cm` is the moving-target distance.

## If you see nothing / garbage
- **Garbage characters** = wrong baud. It must be **115200**.
- **No device** = it's an OTG/cable issue, not the dongle. Confirm the cable carries data (not charge-only) and the OTG adapter is seated. Some phones need OTG enabled in Settings.
- **Connects but silent** = unplug/replug so Android re-grants USB access, then Connect again.

## Notes
- This is the "plug into phone" mobile-sweep path from build 04.
- iPhone won't work (no third-party USB serial).
- Powered straight off the phone's port — fine for C5 + LD2420.
