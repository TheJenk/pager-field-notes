# Theme firmware notes & gotchas

Things learned testing themes on real hardware. Current reference: firmware **2.0.0**, theme framework
**0.9**. Notes below the 2.0.0 section were found on 1.1.x and still apply unless marked.

## Firmware 2.0.0 / theme framework 0.9
2.0.0 expects **`"theme_framework_version": "0.9"`** and adds new screens. Tested with a 0.8 theme:

- **Two warnings at every UI start:** "this theme is missing required components" (they're taken
  from the default theme; pressing back five times on the dashboard opens a navigation safe mode),
  then "this firmware runs a different theme engine version".
- **Borrowed screens have no status bar.** The missing screens come from the stock theme, but
  their status bars are looked up as `fallback_default`, `fallback_network`, `fallback_gps`,
  `fallback_min` and `fallback_bt_recon`, which your theme doesn't define. The log says the screen
  "will draw without a status bar".
- Everything else keeps working.

**New slots a 2.0 theme should style itself:**
- `bt_recon_list_path` (top level): the Bluetooth results list, a slot `template` like the Wi-Fi
  recon list. Variables `$_BT_RESULT_*` (`TITLE`, `MAC`, `RSSI`, `TYPE`, `VENDOR_SHORT`,
  `LAST_SEEN`) and `$_BT_LIST_*` paging.
- `generic_menus`:
  - Bluetooth device screens: `bt_recon_selected_device`, `bt_recon_advertising_data`
    (`$_BT_SELECTED_*`)
  - `recon_new_scan_submenu`
  - `settings_bluetooth_submenu` (`$_BT_*` scan settings)
  - `settings_gps_advanced` (`$_GPS_*`: native/gpsd, connection, protocol, readiness, accuracy,
    satellites, signal)
  - one help page per settings menu: `alerts_dashboard_explained`,
    `settings_{display,general,network,ringtone,system,bluetooth}_submenu_explained`,
    `settings_gps_advanced_explained` and `pineap_{pineap,openap,evilwpa,filters,ssidpool}_
    submenu_explained`
- `status_bars.bt_recon` (items Time, Battery, GPS, Wigle, `BTDB` on/off, and `BTScan` with
  `off`/`both`/`both_active`/`classic`/`le`/`le_active`). Point the recon dashboard at it with
  `"bt_status_bar": "bt_recon"`. Stock also adds `BTScan` to the recon status bar.
- `signal_templates.bt_signal`: thresholds `min_threshold -95`, `weak_threshold -80`,
  `strong_threshold -60`, `max_threshold -40`, with layer lists `minimum_signal`, `weak_signal`,
  `strong_signal` and `max_signal`.

**GPS status icon:** states are now `disconnected`, `acquiring`, `nofix`, `weak` and `fix`. Tested
with a receiver attached and no fix: with all names defined it drew `weak`, without `weak` it
drew `nofix`, and with only the old `on`/`off` it drew `off`. Define all five; keep `on`/`off`
too if the theme also targets 1.1.x.

**Smaller changes:**
- The payload log's error indicator key is now `payload_errored_indicator`.
- Optional layout keys `channel_x`, `channel_y` and `channel_row_height` place the recon
  dashboard's live channel view.
- The legacy top-level `*_path` keys for settings and PineAP menus are no longer read.
- Unused stock template entries were dropped; your own entries still work.
- 2.0 also guards against several theme crashes (empty menu pages, undefined status bars,
  `line_height`).

Same rule as always: on the new screens, only display variables the stock 2.0 screens display.


## Framework version (1.1.2 → 1.1.3)
- 1.1.3 expects **`"theme_framework_version": "0.8"`**. A theme left on `"0.7"` throws a framework
  error on 1.1.3 — bump it.
- 1.1.3 added a **client-mode Wi-Fi status icon** (`WifiClient`) with a 3-state layer map
  (`disabled` / `connected` / `disconnected`) and its own assets, plus a new quick-settings menu that
  the power menu's first row opens. Otherwise the theme-structural keys are the same as 1.1.2.
- **Status-icon sizing:** the stock status glyphs are about **20 px** (e.g. mute ~20×20, volume ~17×20,
  battery ~38×20). Custom status icons should be ~19–20 px tall or they look undersized next to the
  stock ones; a too-wide custom battery will crowd the clock.

## Fonts are wider on the device than in a desktop viewer
Pager fonts are fixed-width, but render **wider on hardware**. A monospace desktop preview advances
~7.83 px per small character (13 px tall); on hardware small text measured **~9.0–9.3 px/char**.

- Sizing `max_chars` at ~7.83 is fine — the firmware truncates long text with "…", which reads well.
- Text positioned right after other text (`x = start + len × width`) must use **~9.2**, or it overlaps
  on the device even though it looked fine in a viewer.
- Fixed text in a tight box (pill, sheet, card) needs slack for ~9.2 px/char, or use `center_text_within`
  with room to spare.
- There is no marquee/scrolling text; long text can only be truncated or wrapped.

## Crash and slowdown traps (seen on hardware)
- **Only display variables the stock theme displays.** Showing an option/editor item's own `variable`
  (volume, theme, timezone, GPS device, ringtone names, wizard SSIDs/passphrases, etc.) crashed the UI
  at boot: generic menus are pre-rendered, and an empty list gives an "index out of range [0] with
  length 0" and a boot-loop.
- **Settings / PineAP home previews:** when a category is highlighted, show only what the stock preview
  shows (a few toggles, date/time, firmware version). Live readings (uptime, battery, temperature,
  storage, IP, GPS) are fetched on every highlight move and make the menu sluggish; show "…" like stock.
- The firmware draws some text in its own colour (white): the keyboard input line, and the battery
  percentage inside the "discharging" battery icon. On a light theme, give those a dark backing.

## Alerts / payload menus
Payload menus shown from DuckyScript `PROMPT` / `ALERT` render as a **modal alert**
(`alert_dialog_path` → the alert-info dialog), **not** the edit/number dialog. A long option list
(e.g. 13 lines) clips unless you enlarge the alert/notification panel and give the input area a high
`max_lines`. Size those panels generously if payloads will show long menus.

**`$_INPUT_NAME` shows the wrong folder name** — *verified (1.1.x)*. Inside a payload category list it
holds the *first payload folder with content* (e.g. "games"), not the folder the user opened. The
stock theme shows the same wrong name. Don't use it as a folder title; use fixed text instead.

## Image economy
Every image is decoded and cached on the device; generated canvases are cached on top. What matters is
**decoded pixels** (sum of width × height of every PNG) and the count of full-screen images, more than
file size.

- The stock theme is ~5.9 Mpx across ~160 images; aim at or below that.
- Share one base background per layout and put the differences (header icon, highlight bar, card, row
  panel) on top as small, reused images.
- Prefer Pager text over baked-in labels (one panel image per button size + the label as text beats an
  on/off image pair per item).
- Crop transparent margins and shift the layer by the crop offset. Full-screen images the firmware
  places itself (`*_path` keys) are never cropped.
- Compress generated flat art hard (e.g. pngquant ≤ 64 colours); boot animations want 5–8 unique frames
  with a short hold, not 15+.

## The big one: you can't launch a payload or run a command from a theme item
On 1.1.3 a theme item's `target` is dispatched only as a component/screen navigation; targets that are
*supposed* to run a command or launch a payload resolve to "component does not exist" and do nothing.
See [firmware findings](../firmware/findings.md) — don't design a theme around launching a payload from
an icon on current firmware.
