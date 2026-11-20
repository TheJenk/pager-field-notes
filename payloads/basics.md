# Payload basics

A payload is a folder containing `payload.sh` (the entry point, with a header comment) plus any
resources. Payloads run as root. They can be plain shell, or drive an on-screen UI by taking over
the framebuffer.

> For a single-file, AI-ready version of everything on these pages (with starter apps and helper
> scripts), see [Pager Payload Skills](../skills/pager-payload-skills.md).

## Where payloads live
| Repo path → device path | Runs when | Extra input |
|---|---|---|
| `library/user/<category>/<Name>` → `/root/payloads/user/…` | picked in the **Payloads** menu | — |
| `library/recon/access_point/<Name>` → `/root/payloads/recon/access_point/…` | run on an **AP selected in Recon** | `$_RECON_SELECTED_AP_*` |
| `library/recon/client/<Name>` → `/root/payloads/recon/client/…` | run on a **client selected in Recon** | `$_RECON_SELECTED_CLIENT_*` plus its AP's `$_RECON_SELECTED_AP_*` |
| `library/alerts/<event>/<Name>` → `/root/payloads/alerts/<event>/…` | a daemon **event** fires (headless) | `$_ALERT` and `$_ALERT_*` |

User categories that exist upstream: `general`, `reconnaissance`, `exfiltration`, `interception`,
`incident_response`, `remote_access`, `prank`, `games`, `evil_portal`, `virtual_pager` and
`examples`. Pick an existing one, and use no spaces in folder names.

## `payload.sh` header
Begin with comments giving at least Title and Author; add Description, Version, Category:
```sh
#!/bin/bash
# Title: Example
# Author: You
# Description: What it does.
# Version: 1.0
# Category: general
```
Use **placeholders** for anything user-specific (URLs, keys, SSIDs). Never ship personal values.

## How a menu-launched payload runs — *verified (2.0.0)*
- The UI **copies `payload.sh` to `/tmp/payload-<n>.sh` and runs the copy**, so `$0` and
  `dirname "$0"` point at `/tmp`, not at your folder. Find your bundled files through
  **`$_PAYLOAD_HOME`** (the payload's real directory), with a hard-coded install path as a fallback.
- `PATH` is `/usr/bin:/bin:/usr/sbin:/sbin`. Tools you keep elsewhere need an explicit path or a
  `PATH` addition.
- The shell is **bash 5.2**, so `${x^^}`, associative arrays and `$EPOCHSECONDS` all work.
- `/lib/hak5/pineapple.sh` defines the dialog result constants:
  `DUCKYSCRIPT_USER_DENIED=0`, `DUCKYSCRIPT_USER_CONFIRMED=1`, `DUCKYSCRIPT_CANCELLED=2`,
  `DUCKYSCRIPT_REJECTED=3`, `DUCKYSCRIPT_ERROR=4`.

## Recon payloads — *from the stock examples*
These run against whatever the user selected in Recon.
- **AP variables:** `_RECON_SELECTED_AP_` + `BSSID`, `SSID`, `OUI`, `HIDDEN`, `CHANNEL`, `FREQ`,
  `ENCRYPTION_TYPE`, `CLIENT_COUNT`, `RSSI`, `PACKETS`, `TIMESTAMP`, `MAC_ADDRESS`,
  `BEACONED_SSID(S)`, `RESPONDED_SSID(S)` and `PROBED_SSID(S)`.
- **Client variables:** `_RECON_SELECTED_CLIENT_` + `MAC_ADDRESS`, `SSID`, `BSSID`, `OUI`,
  `CHANNEL`, `FREQ`, `ENCRYPTION_TYPE`, `RSSI`, `PACKETS`, `TIMESTAMP` and `PROBED_SSID(S)`.

Values can carry stray whitespace, so trim them before matching.

## Alert payloads
The daemon runs them on a **fixed set of four Wi-Fi events**. You cannot add an event type
(e.g. "GPS disconnected", or anything Bluetooth). Only one payload per event is active, chosen in the
Alerts dashboard, and an event only fires if the matching capture feature is enabled. They run
headless.

| Event folder | Fires when | Variables (each continues the prefix) |
|---|---|---|
| `handshake_captured` | WPA handshake / PMKID saved | `_ALERT_HANDSHAKE_` + `SUMMARY`, `AP_MAC_ADDRESS`, `CLIENT_MAC_ADDRESS`, `TYPE`, `COMPLETE`, `CRACKABLE`, `PCAP_PATH`, `HASHCAT_PATH` |
| `deauth_flood_detected` | deauth/disassoc flood | `_ALERT_DENIAL_` + `MESSAGE`, `SOURCE_MAC_ADDRESS`, `DESTINATION_MAC_ADDRESS`, `AP_MAC_ADDRESS`, `CLIENT_MAC_ADDRESS` |
| `pineapple_auth_captured` | enterprise credentials | `_ALERT_AUTH_` + `SUMMARY`, `TYPE`, `USERNAME`, `CHALLENGE_IDENTITY` |
| `pineapple_client_connected` | client joins the Pager's AP | `_ALERT_CLIENT_CONNECTED_` + `SUMMARY`, `AP_MAC_ADDRESS`, `CLIENT_MAC_ADDRESS`, `SSID`, `SSID_LENGTH` |

`$_ALERT` holds the event name. Client-connected fires often, so dedupe with a seen-list and a
cooldown. For "alert me when X" where X isn't one of these events, use a
**background watcher** instead (see [patterns](patterns.md)).

## Boot payload
One payload can be set to run once at each boot. It starts after the UI has finished booting (the
app creates `/tmp/boot_complete` and `/tmp/serverbooted`; wait for them if you call the API early).
An always-on job must loop internally.

## Launching programmatically
- **There is no working launch API on 2.0.0.** `payloads/{id}/run_payload` is a leftover route that
  never receives valid IDs on the Pager. A payload is launched from the menus.
- **From a shell on the device**, `cd <dir> && setsid bash payload.sh >/tmp/p.log 2>&1 </dev/null &`
  runs it, and `UI_TAKEOVER` works. Dialogs and `LOG` don't always reach the screen this way, and
  killing such a run mid-takeover can hang the UI's takeover handling until the UI is restarted. Do
  final tests from the Payloads menu.
- **A theme UI element cannot launch a payload** on current firmware. See
  [firmware findings](../firmware/findings.md).

## Installing a payload
- **Terminal:** copy the folder to `/root/payloads/user/<category>/<Name>/` with `scp` over the
  USB-C link (the Pager is at `172.16.52.1`), then `chmod +x` any binaries.
- **Browser (2.0.0):** the Virtual Pager web UI at `http://172.16.52.1:1471` has a **Files** page
  that can create folders and upload files, and a **web terminal**. Uploaded files may lose the
  executable bit, so payloads should `chmod +x` their own binaries at startup.
- **Payload Portal:** installs published payloads. Its download has a hard timeout of about 60 s,
  so large payloads (big README GIFs, for example) can fail over a weak Wi-Fi link.
- The 2.0.0 update **deleted installed user payloads**; loot, themes and other files under `/mmc`
  survived. Keep copies of your payloads.

## On-screen payloads
To draw your own UI, the `payload.sh` wrapper calls `UI_TAKEOVER`, runs a small native or Go
program that writes `/dev/fb0` and reads `/dev/input/event0`, and calls `UI_RELEASE` on exit (via a
`trap`). See [building native & Go tools](building.md).
