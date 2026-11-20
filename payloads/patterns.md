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

## Live display: slow data in shell, fast drawing in a binary
- **Producer** (shell, detached with `setsid`): does the slow work (API calls, `tcpdump`, `iw`, GPS
  polling), parses it with `awk`, and writes a small state file atomically
  (`> /tmp/app.dat.tmp && mv /tmp/app.dat.tmp /tmp/app.dat`), with a generation counter so the
  reader only reparses on change.
- **Renderer** (native/Go): re-reads the file a few times a second and keeps drawing at up to
  20 fps from its last copy. The UI stays smooth even when data arrives every 1–3 s.
- Keep network or API calls **out of the keypress path**. Redraw from cached data on a key, and
  refresh data on a timer.

## Structuring a bigger app
- **One main loop, as a state machine over screens**, rather than nested blocking loops per screen.
- **Derive timers from the clock**: a stopwatch is `now - start` (`$EPOCHSECONDS` in bash). It stays
  correct across screens and never drifts.
- **Exit-code relaunch:** to switch modes that need a clean start, let the binary exit with a
  special code (e.g. 10) and have `payload.sh` loop, flip the mode and relaunch. Persist UI state in
  a `/tmp` file so it survives the relaunch.

## Fork-free bash for anything interactive — *measured*
Each `$(…)`, pipe or external command costs ~15–30 ms on this CPU. In draw or keypress paths, use
`printf -v`, `${var^^}`, `$(<file)` (which doesn't fork), `$EPOCHSECONDS`, and functions that set a
variable instead of echoing. One redraw went from ~1 s to ~90 ms this way.
Also watch for this: `local a=$1 b=$((a+1))` expands `b` **before** `a` is assigned. Split it into
two `local` lines.

## Process control without `pkill` — *verified*
busybox has `pgrep`, `killall` and `timeout`, but **no `pkill` and no `nohup`** (use `setsid`).
- `pgrep -f "<pattern>"` also matches **the shell running the command** (the pattern is in its
  argv). Run over SSH, it kills your own session. Prefer PID files and `$!`, or use the `[x]yz`
  trick (`pgrep -f "[t]cpdump -i wlan1mon"`).
- `pgrep` patterns are regexes, so `+`, `.` and `[` in paths need escaping. An unescaped `+` made a
  cleanup loop silently miss its targets.
- A `while …; do long_command; done` loop only re-checks its stop flag after `long_command` exits.
  Kill the long-runner itself, or run the whole loop under `setsid` and kill the process group.

## WiGLE logging behaviour — *verified (2.0.0)*
- Rows are only written **while there's a GPS fix**, and each AP is logged **once per boot**, so a
  parked Pager stops adding rows.
- **Every `WIGLE_START` creates a new log file**, so check before starting.
- The uci flag `pineapd.@pineapd[0].logwigle` is configuration, **not live state**. Logging is
  active when the daemon holds a `wigle-*.csv` open:
  `ls -l /proc/$(pgrep pineapd | head -1)/fd | grep -o '/[^ ]*wigle-[^ ]*\.csv' | grep -v -- -bt-`
  (the `-bt-` files are Bluetooth logs).
- If your payload didn't start the log, leave it running on exit.

## Internet from a payload
The Pager only reaches the internet through **client-mode Wi-Fi** (`wlan0cli`, **2.4 GHz only**,
sharing the built-in radio). The USB-C link doesn't route internet by default. That link can be
weak and erratic, so use short `curl --max-time` values, retries and cached results. Public APIs
rate-limit; some return an **empty success** when throttled, so treat empty as failure.

## Serial / UART peripherals — *verified*
- **Opening a USB-serial port toggles DTR, and many boards reset on DTR.** Open once and hold the
  descriptor for the whole session (`exec 3<>/dev/ttyUSB0`). Configure with
  `stty … clocal -hupcl raw`. Expect garbage first and sync on a known prompt.
- **The GPS stack can grab your port.** gpsd (1.1.x) or the daemon's GPS autodetect (2.0) may
  probe a new USB-serial device at GPS baud rates, toggling DTR and corrupting traffic. On 1.1.x
  stop gpsd unconditionally (a `pgrep` check races procd's respawn); on 2.0 check the configured GPS
  device isn't yours. A probe at 1200 baud can reboot some microcontroller boards into their
  bootloader.
- In native code keep `VTIME` small (≤0.2 s) and do the waiting in a deadline loop. With `VMIN=0`,
  a large `VTIME` blocks every idle read for the full timeout.
- Repeated open/close cycles plus large transfers over a glitching USB link have crashed a host
  computer's USB stack. Batch serial work, unplug peripherals before large copies, and stop when
  the link starts dropping.

## Testing safely on a device — *verified*
- **Killing a payload mid-`UI_TAKEOVER`** (especially one launched from a shell) can leave the UI's
  takeover handling hung. Exit through the app's own exit path. If the UI is stuck, run
  `/etc/init.d/pineapplepager restart` (recon recovers by itself).
- After the display dims or a modal opens, **injected key events may stop updating the UI**. Raise
  `lcd_timeout`/`dim_timeout` while testing.
- **RTL-SDR:** `rtl_power` only writes on a clean exit, and killing it mid-sweep can wedge the tuner
  until it's replugged. Use one-shot (`-1`) sweeps.
- **Takeover from a child process isn't enough.** Call `UI_TAKEOVER` in `payload.sh` itself. If only
  the binary calls it, the payload log screen keeps redrawing and the desktop flashes through. Draw
  a first frame **before** any slow setup (opening a serial port, a first fetch), and put
  `UI_RELEASE` first in cleanup.

## Editing device files safely
The root filesystem is an overlay (read-only squashfs + read-write overlay). Adding or editing a file
copies it up to the writable overlay: it **persists across reboots**, never touches the signed image,
and is reversible by deleting the overlay copy. A firmware update may revert it, so keep a copy of
anything you add. See [firmware findings](../firmware/findings.md) for a hotplug script fix that uses
exactly this.
