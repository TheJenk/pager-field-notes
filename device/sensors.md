# Sensors & sysfs

## Battery / charging
- `/sys/class/power_supply/bq27546-0/status` → `Charging` / `Discharging` / `Full` / `Not charging`.
- `/sys/class/power_supply/bq27546-0/capacity` → percent (integer).

These are the cleanest way to detect charge state / level from a script (poll and act on changes).

## Buzzer (tones / ringtones)
- `/sys/class/leds/buzzer/frequency` and `/sys/class/leds/buzzer/brightness`.
- It's a **monophonic square-wave** tone generator — one frequency at a time. This is how RTTTL
  ringtones play. Any "audio" on the device is a single melodic line of tones, not PCM playback
  (there's no ALSA device).
- *Verified:* `frequency` is the pitch in Hz, and `brightness` (0–255) is the duty cycle (255 ≈ 50 %,
  the loudest; 0 = silent). Writing the `volume` file does nothing.
- **Direct sysfs writes ignore the user's mute**, while the built-in `RINGTONE` honours it. If a
  payload plays its own sounds, give it an on/off setting, and **always write `0` to `brightness` on
  exit** (in the cleanup `trap` too) or a note keeps sounding. Time notes with a millisecond clock,
  not frame counts.

## Temperature
- There is **no readable `/sys/class/thermal` zone** (no SoC temperature). The firmware manages
  overheat/shutdown itself.
- What *is* readable: the **battery temperature** at `/sys/class/power_supply/bq27546-0/temp` (tenths
  of °C), and the **Wi-Fi radio temperature** at `/sys/class/hwmon/hwmon*/temp1_input` (m°C).

## Display backlight
- `/sys/class/backlight/backlight_pwm/brightness` (0–16). The user's brightness and dim/sleep
  timeouts are also in the preferences API (`pager/preferences/get` and `set`).

## GPS
**2.0.0+:** PineAP reads the receiver natively and gpsd is disabled. Use the local API route
`pineap/gps/get` (JSON with `lat`/`lon`/`fix`/`connected`/accuracy/satellites), or `GPS_GET`. See
[What changed in 2.0.0](../firmware/2.0.0.md). The gpsd notes below apply to 1.1.x, or to 2.0
with gpsd switched back on.

### gpsd (1.1.x)
- `gpsd` (3.25) runs against the configured device, which you can read with
  `uci -q get gpsd.core.device` — it's a `/dev/serial/by-path/<id>` symlink, not a raw `ttyUSB`/`ttyACM`.
- Read live data:
  - `gpspipe -w` → JSON. `TPV` reports have `mode` (1 = no fix, 2 = 2D, 3 = 3D) plus `lat`/`lon`/`speed`
    when fixed. `SKY` reports carry **`hdop`** (horizontal dilution of precision — a reliable accuracy
    metric, lower is better; `uSat`/`nSat` satellite counts are **not** always present).
  - `gpspipe -r` → raw NMEA sentences.
- gpsd emits roughly **1–2 reports/second whenever the receiver is connected, even with no fix** — so
  "data has stopped for a few seconds" is a reliable "GPS is gone" signal (see
  [patterns](../payloads/patterns.md) for robust detection, and why the stock `GPS_LIST` can't be
  trusted for this).
