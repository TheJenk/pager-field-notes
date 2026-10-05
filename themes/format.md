# Theme format

A theme is a folder with `theme.json`, a `components/` directory of screen JSON, and an `assets/`
directory of PNGs, for the **480 × 222** screen. Install by copying the folder under `/root/themes/`
(or `/mmc/root/themes/`) and selecting it in **Settings → General → Theme**. The **stock theme** that
ships on the device is the reference for how every screen works; the practical way to build one is a
generator that starts from the stock theme and restyles each screen.

## `theme.json`
Maps **slots** to component files and defines shared resources. Keep every stock key:

- `generic_menus.*` — most screens (dashboards, submenus, wizards). This is a **registry**: a custom
  screen is only a valid navigation target if it's listed here.
- `*_path` keys — special screens and full-screen images (launch dialog, spinner, lock screen,
  keyboards, alerts, boot animation, etc.).
- `option_dialogs` — the value-picker dialogs (another registry; accepts added keys).
- `recon_lists`, `status_bars` — recon tables and the status-bar variants (`default`, `network`,
  `recon`, `min`, `gps`).
- Template groups — `string_templates`, `toggle_templates`, `radio_templates`, `signal_templates`.
- `color_palette` — named colours looked up by name. Keeping stock names and giving them roles (e.g.
  one = body text, one = accent, one = text-on-accent) lets stock templates/screens work unchanged.
- `theme_version`, `theme_framework_version`, `author`.
- `_unused` — holds `commands`, `payload_shortcuts`, `simple_images`. **Note:** despite the name the
  firmware *does* read `_unused` for these; but see [firmware findings](../firmware/findings.md) — the
  command/shortcut *targets* don't actually fire on current firmware.

## A screen (component JSON)
- `background.layers`, `menu_items` (or `pages[].menu_items`), optional `button_map` and `status_bar`.
- **Template lists** (`"template": {...}`) have `Slot0..SlotN` items the firmware fills from a list —
  keep the stock slot count.
- **Layers:** `image_path` (+ `x`, `y`, optional `recolor_palette` tint), `text` (+ `text_size`
  small/medium/large, `text_color_palette`), `variable_name` with `string_template` or `use_template`,
  `toggle_template` (`enabled`/`disabled` layer lists), `center_text_within`
  (`start_x`/`end_x`/`start_y`/`end_y`), `max_chars`, `wrap_text`, `max_lines`.
- **Items:** `layers` (normal) and `selected_layers` (focused), a `target`, optional `variable`,
  `unselectable`, and a `button_map` (`a`/`b`/`up`/`down`/`left`/`right` →
  `select`/`back`/`next`/`previous`/`noop`/`goto <item id>`).

## What a `target` can be
A target is resolved as a **component/screen name to navigate to**, or one of a fixed set of built-in
actions:

- an existing screen/component name (go to it) — must be registered (see registries above),
- `function_*` built-in actions (sleep screen, shutdown, lock, toggles, etc.),
- `inline_toggle` (+ a boolean `variable`) — flips it in place,
- `option_dialog*` (+ a `variable`) — opens a value picker that applies on select,
- `edit_*_dialog`, `launch_payload` / `launch_payload_dialog` (the payload-browser confirm flow),
- `power_menu`, and `goto <id>` to move focus (including to another page).

Anything the firmware doesn't recognize logs **`Theme targets component that does not exist: <target>`**
and does nothing. See [firmware findings](../firmware/findings.md) for targets that *look* like they
should run a command/payload but don't on current firmware.

## Other format facts
- `windowed_canvas: true` draws over the previous screen, so small cards/sheets/overlays can leave the
  rest transparent.
- Item `animation` (+ `animation_interval`) works on at least the launch dialog and the spinner.
  `boot_animation.json` is a list of full-screen frames looped during boot.
- The stock status bar draws its own icons at the top-right (roughly x 250–480); design headers around
  it, and its `background_color` is not painted.
