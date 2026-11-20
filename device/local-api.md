# Local API & Helpers

## Local HTTP API over a unix socket
The device exposes a local HTTP API on a unix socket:

```sh
curl --unix-socket /tmp/api.sock http://localhost/api/<route>
```

Useful routes seen in use:

- `pager/preferences/get` — read preferences (volume, vibrate, brightness, clock mode, theme name,
  ringtones, timeouts…).
- `pager/preferences/set` — **PUT** a *partial* JSON body to change prefs, e.g.
  `curl --unix-socket /tmp/api.sock -X PUT -d '{"volume":0,"vibrate":false}' http://localhost/api/pager/preferences/set`.
  (It's **PUT**; a POST returns 405.)
- `system/serial/list` — the serial-device list the UI shows (note: this is **cached** and can go
  stale after an unplug — see [firmware findings](../firmware/findings.md)).
- `payloads/{id}/run_payload` — a leftover route from older Pineapple firmware. *Verified on 2.0.0:*
  it **doesn't launch anything** on the Pager, because it expects numeric IDs from a list the Pager
  never fills. There's no working launch API; payloads are launched from the menus.
- `pineap/get_config` (logging flags such as `loghandshake`, `logpcap`, `logwigle`, plus loot
  paths) and `pineap/hostapd/get_config` (AP state). These flags gate which alert events can fire.
- Various other `pineap/*` routes for Wi-Fi/recon config.
- *Seen in the firmware but not fully verified:* `pager/debug/input/emulatekeys` (key injection),
  `payloads/stop_payload`, `pager/display/screen`, and `files/raw/<path>` and `files/zip/<path>`
  (used by the web UI's file browser).
- **2.0.0+:** `pineap/gps/get` (native GPS position and status), `recon/wifi/{status,devices,count}`,
  `recon/wifi/devices/{mac}/clients`, `recon/bluetooth/{status,devices}`,
  `recon/bluetooth/devices/{mac}` and `recon/setup`. See [What changed in 2.0.0](../firmware/2.0.0.md).

Shell helpers wrapping the API live in `/lib/hak5/commands.sh` (`HAK5_API_GET` / `HAK5_API_POST` /
`HAK5_API_PUT`). The HTTP method matters: a wrong one returns **405**.

## Virtual Pager web UI (2.0.0)
The device serves a browser UI at **`http://172.16.52.1:1471`** over the USB-C link (and, if
enabled in Settings, over client-mode Wi-Fi). Besides mirroring the screen, the 2.0.0 version has:
- **recon tables** and a WiGLE map;
- a **Files** page that browses the filesystem and can **upload** files (drag and drop into an open
  folder). This gives a way to install payloads without a terminal. Uploaded files may lose the
  executable bit, so payloads should `chmod +x` their own binaries.
- a **web terminal**, so users without SSH can still run commands such as `logread`.

## Payload helper binaries and functions
Available to payloads (in `/usr/bin` unless noted):

- **`LOG`** — write a coloured line to the payload log.
- **`ALERT`** — `ALERT "message"` raises a **modal alert popup** on screen (returns 0; it does **not**
  write to `logread`, so add your own `logger` line if you want an audit trail).
- **`ALERT_RINGTONE`** — play the alert ringtone.
- **`UI_TAKEOVER` / `UI_RELEASE`** — functions exported into the payload runtime that tell the UI to
  release / reclaim the framebuffer around an on-screen payload. Always `UI_RELEASE` on every exit path
  (use a shell `trap`) or the UI and your payload will fight over `/dev/fb0`.

Also present and handy: `setsid`, `logger` (writes to `logread`; tag with `-t <name>`), `gpspipe`,
`gpsctl`, `gpsd`, `awk`, `tcpdump`, `iw`, `curl`, `uci`, `stty`. 2.0.0 adds `blue5ctl` and ships
`sqlite3`. busybox `awk` has `systime()`/`strftime()`. On 2.0 gpsd is off by default, so
`gpspipe` returns nothing.

**Not present:** `python`, and no serial terminal program (`microcom`/`picocom`/`screen`/`cu`) — talk
to serial devices with `stty` + redirection or a small custom program.
