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
- `payloads/{id}/run_payload` — run a payload by numeric ID (the UI's own launch path).
- Various `pineap/*` routes for Wi-Fi/recon config.

Shell helpers wrapping the API live in `/lib/hak5/commands.sh` (`HAK5_API_GET` / `HAK5_API_POST` /
`HAK5_API_PUT`).

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
`gpsctl`, `gpsd`, `awk`, `tcpdump`, `iw`, `curl`, `uci`, `stty`.

**Not present:** `python`, and no serial terminal program (`microcom`/`picocom`/`screen`/`cu`) — talk
to serial devices with `stty` + redirection or a small custom program.
