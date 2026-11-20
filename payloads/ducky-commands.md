# Hak5 command reference (DuckyScript verbs)

The Pager's DuckyScript "commands" are the all-caps names in `/usr/bin`. Almost all of them are
symlinks to **one multicall binary** that turns each call into a request to the local API
(`/tmp/api.sock`); that's why dialogs are drawn by the Pager app and the result comes back on
stdout. You call them straight from a `payload.sh`. There are **139** on firmware 1.1.3, and 2.0.0
has the same set (no new commands). This page maps what's built in so you don't reimplement it. (The
official docs describe usage; this is the inventory, grouped.)

> Payloads also get shell helper functions from `/lib/hak5` (`commands.sh`, `pineapple.sh`) and the
> `UI_TAKEOVER`/`UI_RELEASE` handoff. See [local API & helpers](../device/local-api.md).

## On-screen UI & interaction
`ALERT`, `CONFIRMATION_DIALOG`, `ERROR_DIALOG`, `PROMPT`, `PROMPT_IMAGE`, `PROMPT_QR`,
`LIST_PICKER`, `TEXT_PICKER`, `NUMBER_PICKER`, `IP_PICKER`, `MAC_PICKER`, `PASSWORD`,
`WAIT_FOR_BUTTON_PRESS`, `WAIT_FOR_INPUT`, `START_SPINNER`, `STOP_SPINNER`, `LOG`,
`UI_TAKEOVER`, `UI_RELEASE`, `ENABLE_DISPLAY`, `DISABLE_DISPLAY`, `RESTART_UI`.

## Feedback (LEDs, buzzer, haptics)
`LED`, `DPADLED`, `DPADLED_CONFIG`, `VIBRATE`, `RINGTONE`, `DO_A_BARREL_ROLL`.

## Device info
`BATTERY_PERCENT`, `BATTERY_CHARGING`, `GPS_GET`, `GPS_LIST`, `GPS_CONFIGURE`, `FIND_CLIENT_IP`.

## Wi-Fi recon / PineAP (the `PINEAPPLE_*` family)
- Recon: `PINEAPPLE_RECON_NEW`, `PINEAPPLE_HOPPING_START`, `PINEAPPLE_HOPPING_STOP`,
  `PINEAPPLE_SET_BANDS`, `PINEAPPLE_EXAMINE_BSSID`, `PINEAPPLE_EXAMINE_CHANNEL`,
  `PINEAPPLE_EXAMINE_RESET`.
- Deauth: `PINEAPPLE_DEAUTH_CLIENT`, `PINEAPPLE_DEAUTH_CLIENT_ALL`, `PINEAPPLE_DEAUTH_CLIENT_CSA`.
- Impersonation: `PINEAPPLE_MIMIC_ENABLE`, `PINEAPPLE_MIMIC_DISABLE`.
- Filters (device/MAC/SSID/network): `PINEAPPLE_{DEVICE,MAC,SSID,NETWORK}_FILTER_{ADD,ADD_FILE,
  DELETE,CLEAR,LIST,MODE}`.
- SSID pool: `PINEAPPLE_SSID_POOL_{ADD,ADD_FILE,DELETE,CLEAR,LIST,START,STOP,COLLECT_START,
  COLLECT_STOP}`.
- Loot: `PINEAPPLE_LOOT_ARCHIVE`.

## Wi-Fi AP / client / capture
`WIFI_CONNECT`, `WIFI_DISCONNECT`, `WIFI_CLEAR`, `WIFI_WAIT`,
`WIFI_MGMT_AP` (+ `_CLEAR`/`_DISABLE`/`_HIDE`), `WIFI_OPEN_AP` (+ `_CLEAR`/`_DISABLE`/`_HIDE`),
`WIFI_WPA_AP` (+ `_CLEAR`/`_DISABLE`/`_HIDE`), `WIFI_PCAP_START`, `WIFI_PCAP_STOP`.

## WiGLE (wardriving)
`WIGLE_LOGIN`, `WIGLE_LOGOUT`, `WIGLE_START`, `WIGLE_STOP`, `WIGLE_UPLOAD`.

## Network services
`SYSTEM_DNS`, `DNSSPOOF_{ENABLE,DISABLE,ADD_HOST,DEL_HOST,CLEAR}`,
`OPENVPN_{CONFIGURE,ENABLE,DISABLE}`, `WIREGUARD_{CONFIGURE,ENABLE,DISABLE}`,
`AUTOSSH_{CONFIGURE,ENABLE,DISABLE,ADD_PORT,CLEAR}`, `SSH_ADD_KNOWN_HOST`.

## USB storage
`USB_STORAGE`, `USB_WAIT`, `USB_EJECT`, `USB_FREE`.

## Payload config & Pager Portal
`PAYLOAD_GET_CONFIG`, `PAYLOAD_SET_CONFIG`, `PAYLOAD_DEL_CONFIG`,
`PAGERPORTAL_PAYLOAD_{INSTALL,LIST,LIST_UPDATES,REFRESH,UPDATE}`.

## System / firmware / theme
`INSTALL_FIRMWARE`, `RESTORE_DEFAULT_THEME`, `DEVELOPER_THEME_RELOAD`, `TOS_ACCEPT`, `SLA_ACCEPT`.

---

Notes:
- Prefer these built-ins over hand-rolled equivalents — they integrate with the UI (dialogs, spinner,
  takeover) and the daemon (recon/PineAP/filters) correctly.
- `ALERT` shows a modal but does **not** write to `logread`; add a `logger -t <tag>` line if you want
  an audit trail (see [patterns](patterns.md)).
- For "alert me when X" where X isn't a built-in alert event, don't look for a command — use a
  background watcher (see [patterns](patterns.md)).

## Usage notes & gotchas — *verified*
- **Results come back on stdout:** `choice=$(LIST_PICKER "Title" opt1 opt2 opt3 default)` (the last
  argument is the default). `CONFIRMATION_DIALOG` prints `1` (yes) or `0` (no); its exit status
  signals rejected (3) or error (4). `START_SPINNER` prints an id for `STOP_SPINNER`.
  `WAIT_FOR_INPUT` prints the pressed button's name.
- **`ALERT` queues behind an open `LIST_PICKER`.** It only appears after the menu closes. Use
  `PROMPT` for feedback inside a menu flow; `ALERT` suits background or task-flow payloads.
- **`PROMPT_QR` and `PROMPT_IMAGE` are modals the app draws**, not a screen takeover: the payload
  blocks until they're dismissed. Images must be **PNG or JPEG**.
- **The dialogs and `WAIT_FOR_INPUT` need the UI.** During `UI_TAKEOVER`, read
  `/dev/input/event0` yourself. `DPADLED`, `VIBRATE` and `RINGTONE` still work during a takeover.
- **Sound follows the user's mute.** With the volume preference at 0, `RINGTONE` and
  `ALERT_RINGTONE` are silent. A second `RINGTONE` is refused while one is playing.
- **Built-in ringtone names:** `alert battery_critical battery_low bonus bootup deskphone error
  getkey getmachine hack_stealth hak5_the_planet halt health leveldone mute noway pager
  power_connected power_disconnected pushwall ringring single_beep urgent warning xp yeah`. You can
  also pass any RTTTL string.
- `DISABLE_DISPLAY`/`ENABLE_DISPLAY` only switch the backlight; they don't pause the UI.
- `UI_TAKEOVER`, `UI_RELEASE`, `DISABLE_DISPLAY`, `ENABLE_DISPLAY`, `DO_A_BARREL_ROLL`,
  `USB_STORAGE` and `USB_WAIT` **ignore `--help` and just run.** Don't loop `--help` over the whole
  set.
- `DO_A_BARREL_ROLL` starts a looping D-pad LED chase; `DPADLED off` (or a UI restart) ends it.
- `GPS_CONFIGURE` changed its arguments in 2.0.0 (see [What changed in 2.0.0](../firmware/2.0.0.md)).
- `PINEAPPLE_SET_BANDS wlan1mon 2` returned success on 2.0.0, but hopping kept visiting 5 GHz.
