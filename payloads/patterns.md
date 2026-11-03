# Proven patterns

## Toggleable background watcher with on-screen alerts
The backbone of every "alert me when X happens" payload (GPS fix/signal, geofence, movement, charger,
battery, storage, USB drive, uplink, SSH login, new loot…). Run the payload once to turn it **on**,
again to turn it **off**. It raises a modal `ALERT` on a transition and logs an audit line; it never
stops anything else.

Key points, with the traps that bite:

- **Detach** the watcher with `setsid "$SELF" __watch &` (re-exec yourself with a sentinel arg); the
  watcher writes its own PID to a pidfile, and re-running the payload sees the live pidfile and stops it.
- **Stop with a process-group kill:** `kill -TERM -"$pid"` (note the leading `-`). Because `setsid`
  makes the watcher a process-group leader, the negative PID kills the watcher **and its `sleep`
  child**, so "off" is instant. A plain `kill "$pid"` only runs the watcher's `trap` *after* its current
  `sleep` finishes — so with a long interval (say 30 s) "off" lags by up to one interval and a quick
  on/off test looks like it failed. Use the group kill, falling back to a plain kill:
  ```sh
  kill -TERM -"$p" 2>/dev/null || kill -TERM "$p" 2>/dev/null
  ```
- **Alert + audit:** `notify(){ ALERT "$1"; logger -t <Tag> "$1"; }` → modal popup **and** a line you
  can review later with `logread | grep <Tag>` (ALERT itself doesn't log).
- **Transition-only:** track state and alert only on the edge (ok→bad), then re-arm on the reverse, so
  you don't fire every tick.
- Put tunables (thresholds, interval, radius) as variables at the top.

## GPS liveness (don't trust `GPS_LIST`)
**2.0.0+:** poll the local API `pineap/gps/get` and use its `connected` and `fix` fields; gpsd is off
by default. What follows is for gpsd on 1.1.x.

- `GPS_LIST` reads the UI's cached serial list; after an unplug it keeps reporting the device. For "is
  the GPS giving data right now", use **gpsd liveness**: stream `gpspipe -w`, refresh a timestamp on
  each `TPV`/`SKY` line, and treat "no data for N seconds" as lost. gpsd emits ~1–2/s even with no fix,
  so a ~5–6 s threshold catches an unplug within seconds without false alarms.
- For "is it physically present", test the by-path node following the symlink:
  `[ -c "$(uci -q get gpsd.core.device)" ]` reads false after an unplug because the real `tty` node is
  gone and the by-path symlink dangles (even though the symlink *file* can linger).
- A robust watcher streams `gpspipe -w` to a file, **respawns** it if it dies (so it reconnects after a
  gpsd restart), and parses the last `TPV`/`SKY` each tick. Signal quality = `hdop` from `SKY`
  (`> ~5` is poor), compared with `awk`. Geofence = `awk` equirectangular distance from a saved home
  point. gpsd does not always auto-resume after a replug — `/etc/init.d/gpsd restart` reattaches it.

## Takeover / peripheral gotchas
- **Bluetooth (2.0.0+):** read the built-in `blue5` scanner (`blue5ctl fetch`) and leave it running;
  starting `bluetoothd` or rescanning yourself breaks the Pager's own BT recon.
- **Wi-Fi (2.0.0+):** sniff passively on the monitor interfaces PineAP already runs
  (`tcpdump -e -n -l type mgt`) instead of adding and retuning your own.
- **Bluetooth (1.1.x):** `btmgmt` / `hciconfig` tend to hang in a payload context (they fight the Bluetooth
  daemon). Use `bluetoothctl` (`power off`/`on`, `--timeout N scan on`) and parse its output.
- **RTL-SDR / `rtl_power`:** it buffers output (only writes on a clean exit), and killing it mid-run can
  **wedge the dongle** until a physical replug. Prefer one-shot sweeps; never SIGKILL it mid-run.
- **Framebuffer handoff:** always `UI_TAKEOVER` before drawing and `UI_RELEASE` on *every* exit path
  (use a `trap`), or the UI and your payload fight over `/dev/fb0`.

## Editing device files safely
The root filesystem is an overlay (read-only squashfs + read-write overlay). Adding or editing a file
copies it up to the writable overlay: it **persists across reboots**, never touches the signed image,
and is reversible by deleting the overlay copy. A firmware update may revert it, so keep a copy of
anything you add. See [firmware findings](../firmware/findings.md) for a hotplug script fix that uses
exactly this.
