# Theme getting started — a practical guide

A start-to-finish path for building a Pager theme, with the lessons that save you from boot-loops,
sluggish menus, and "looks fine in a viewer, broken on the device."

## The mental model
- A theme is `theme.json` + `components/*.json` + `assets/*.png` for a **480 × 222** screen.
- **It's a restyle of the stock theme, not a from-scratch UI.** The stock theme that ships on the
  device already knows how every menu, list, dialog, wizard and status bar works. Start from it and
  change the *look*; keep the *structure*. Touching structure (slot counts, `theme.json` keys, which
  variables a screen shows) is how you crash the device.
- Install = copy the theme folder to `/root/themes/` (or `/mmc/root/themes/`) and pick it in
  **Settings → General → Theme**.

## By hand vs. a generator
You *can* hand-edit the JSON, but a real theme is ~40 screens plus every list/dialog, and each must
keep its menus and D-pad working and stay within the image budget. Editing all of that by hand is slow
and easy to break. The sustainable approach is a **generator**: code that starts from the stock theme,
rewrites each screen's look, and draws every image. See
[Building a theme generator](building-a-generator.md). The rest of this page applies either way.

## The build / test loop
1. **Generate/edit** the theme.
2. **Fix navigation** — every D-pad direction must move the way the screen looks (see below).
3. **Check** for crash-prone variables, broken navigation, missing files, palette issues, and the
   image budget *before* putting it on hardware.
4. **Preview on your computer** — render each screen to PNG (a host renderer) so you can review layout
   and contrast without the device.
5. **Deploy to the device** and look at it for real:
   - Copy the folder to `/root/themes/<name>/` (e.g. `tar | ssh … tar x`), then reload the UI
     (`/etc/init.d/pineapplepager restart`) or re-select the theme.
   - **Capture the screen** to compare against your preview: read `/dev/fb0` (222×480 RGB565) and
     reshape with the rotation `fb[lx*222 + (221-ly)]` → a 480×222 image. This catches font-width and
     status-bar overlaps a desktop viewer hides.
   - After a UI restart a "Welcome" modal can appear and eat button presses — dismiss it first.
6. **Iterate.** Palette and layout tweaks are cheap; keep cycling preview → device.

## Lessons that save pain
(Details in [theme format](format.md) and [firmware notes & gotchas](firmware-notes.md); the
highlights:)

- **Set `theme_framework_version` to `0.8`** for current firmware (an old `0.7` errors).
- **Only display the variables the stock theme displays.** Showing an option/editor item's own
  `variable` (volume, theme, timezone, GPS device, ringtone names, wizard SSIDs/passphrases…) crashes
  the UI at boot (empty-list "index out of range" → boot-loop). This is the #1 way themes break.
- **Keep home/Settings previews light** — show the same few values the stock preview shows, not live
  uptime/battery/temp/IP/GPS, which are re-fetched on every highlight move and make menus sluggish.
- **Fonts are wider on hardware** (~9.2 px/char small) than in a monospace viewer (~7.83). Leave slack
  in fixed boxes, or text that's positioned right after other text will overlap on the device. There's
  no scrolling text — truncate (the firmware adds "…") or wrap.
- **Respect the image budget.** Every PNG is decoded and cached; aim at/below the stock theme's
  ~5.9 Mpx. Share one base background per layout and layer small reused pieces on top; prefer Pager
  **text** over baked-in label images; crop transparent margins and compress hard.
- **Navigation must match the layout.** Side-by-side buttons use left/right; grids use all four
  directions; `goto <item id>` jumps focus (use short ids). Audit it, don't eyeball it.
- **Design around the firmware status bar** (top-right, ~x 250–480), and remember the firmware draws a
  few things in its own white (keyboard input line, battery % inside the icon) — on a light theme, put
  a dark backing behind those.
- **Payload menus** shown via `PROMPT`/`ALERT` render as a modal alert — size that panel generously or
  long option lists clip.
- **You can't launch a payload or run a command from a theme item** on current firmware — don't design
  around it.

## Variants (light / dark)
Define colours as **named tokens** and keep the layout identical between variants; a single generator
can emit both a light and a dark theme by swapping the palette (drive it with an env flag). Design
changes then land in both at once.

## Publishing
- One `README.md` per theme: name, author, description/inspiration, known limitations, and the
  **firmware version** you built for.
- **Original art only.** No copyrighted material or branding that impersonates another product; use
  openly-licensed fonts; describe inspiration generically rather than naming trademarked products.
- Test on a real Pager (alerts, recon, payload launch) before submitting a pull request.
