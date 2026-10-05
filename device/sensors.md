# Sensors & sysfs

## Battery / charging
- `/sys/class/power_supply/bq27546-0/status` → `Charging` / `Discharging` / `Full` / `Not charging`.
- `/sys/class/power_supply/bq27546-0/capacity` → percent (integer).

These are the cleanest way to detect charge state / level from a script (poll and act on changes).

## Buzzer (tones / ringtones)
- `/sys/class/leds/buzzer/frequency` and `/sys/class/leds/buzzer/brightness`.
- It's a **monophonic square-wave** tone generator — one frequency at a time. This is how RTTTL
  ringtones play. Any "audio" on the device is a single melodic line of tones, not PCM playback.

## Temperature
- There is **no readable `/sys/class/thermal` zone**. The firmware manages overheat/shutdown itself.
  Don't try to build a temperature alert off sysfs — there's nothing to read.

## GPS (gpsd)
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
