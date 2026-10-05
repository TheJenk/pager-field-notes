# Building a theme generator

A theme is ~40 screens plus every list/dialog/wizard. Editing that by hand and keeping it correct is
painful. The reliable approach is a **generator**: a small program that starts from the stock theme
(which already knows how every menu works), rewrites each screen's *look*, and draws every image —
reproducibly, every time you run it. This is how larger/maintained themes are built.

## Why generate
- **Correctness:** by restyling the stock components you keep every menu, list, dialog and status bar
  working; you never hand-maintain slot counts or lose a `theme.json` key.
- **Consistency:** all art comes from one place (e.g. SVG), so colours/metrics stay uniform and a
  palette change re-themes everything.
- **Variants:** one generator can emit light + dark (or more) from the same layout by swapping the
  palette.
- **Safety nets:** generation can run checks (crash variables, navigation, image budget) and previews
  on every build.

## Shape of a generator (a proven layout)
Two main stages plus a build pipeline:

### 1. Config + base
- A small config: theme name, author, output paths, and the **palette** (named colour tokens).
- The **stock theme** as the base input — always read from a clean copy so re-running is deterministic.

### 2. Components stage (the JSON)
Walk the stock components and rewrite each screen's look in code:
- swap backgrounds, selection/row bars, titles, dialog/sheet chrome, button art;
- recolour via palette tokens; set `text_color_palette`/`text_size`;
- **keep the structure:** stock slot counts in template lists, every `theme.json` key, and only the
  variables the stock screen shows (see [firmware notes](firmware-notes.md) — showing the wrong
  variable boot-loops the device);
- **register any custom screen** you add in the right `theme.json` registry (`generic_menus` for
  dashboards/menus, `option_dialogs` for value pickers), or its target won't resolve.
Write the dashboard/home layout here too (it's just another component).

### 3. Art stage (the PNGs)
- Define each asset as a vector (e.g. an **SVG** string) and **render it to PNG** — a headless browser
  (Chromium) or an SVG rasteriser works; render at 1× device pixels.
- **Recolour** any stock PNGs you reuse instead of redrawing them.
- **Crop** transparent margins to the smallest box and shift the referencing layer by the crop offset
  (full-screen images the firmware positions itself, the `*_path` keys, are *not* cropped).
- **Compress** (e.g. pngquant, ≤ 64 colours for flat art), **prune** images nothing references, and
  **enforce the pixel budget** (aim ≤ the stock ~5.9 Mpx of decoded pixels).

### 4. Build pipeline
Run the stages in order, then finish with:
- **Navigation fix:** programmatically make every D-pad direction match the layout and wire `goto`
  jumps — don't hand-maintain button maps across dozens of screens.
- **Checks:** crash-prone variables, navigation reachability, missing files, palette/token use, image
  budget, README presence.
- **Preview:** render every screen to PNG on your computer for review.
- Wrap it as `build.sh` (optionally `build.sh light|dark|all`).

## Developer ergonomics (what makes it pleasant)
- **Host preview** — the single biggest time-saver: render screens to PNG on your computer so you
  iterate layout/contrast without flashing to the device every time.
- **On-device loop** — a one-liner to `tar`/`scp` the built theme to `/root/themes/<name>/`, restart
  the UI, and grab `/dev/fb0` to compare against the preview (catches hardware-only issues like font
  width and status-bar overlap).
- **Palette as tokens** — everything references named colours, so light/dark variants share one layout
  and a retheme is a palette edit.
- **Theme-specific names** — name generated files after your theme, not after the stock theme, so the
  output reads cleanly.
- **Deterministic** — always regenerate from a clean stock copy; never edit the output by hand (edit
  the generator and re-run), so builds are reproducible and reviewable.

## Minimal build order, summarised
```
config + stock copy
   → components (restyle JSON, keep structure, register custom screens)
   → art (SVG→PNG, recolour, crop, compress, prune, budget)
   → dashboard/home JSON
   → navigation fix
   → checks  → preview  → (deploy to device + capture to verify)
```

With this in place, making a new theme is mostly: pick a palette and a layout language, point the art
stage at your new look, and let the pipeline keep every screen correct.
