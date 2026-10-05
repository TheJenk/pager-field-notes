# Building native & Go tools

For anything with an on-screen UI, shell shells out to a compiled program that takes over the
framebuffer. Three realistic choices.

## Native freestanding-MIPS C (tiny footprint)
Produces a ~20 KB static binary, zero dependencies, instant start, full framebuffer control.

- **Toolchain:** a `mipsel-unknown-linux-gnu` GCC cross-compiler.
- **Flags:** `-O2 -fno-pic -mno-abicalls -G0 -nostdlib -ffreestanding -static -s -Wl,-e,_start`.
- **No libc:** make raw **o32 syscalls** inline; stay **integer-only** (no FPU). Provide your own
  `memcpy`/string helpers and a small bitmap font.
- **Screen:** `mmap` `/dev/fb0` and write RGB565 with the rotation `fb[lx*222 + (221-ly)]` (see
  [display & input](../device/display-and-input.md)); redraw at ~20 fps or on input.
- **Input:** read `struct input_event` from `/dev/input/event0`; handle the Pager key codes (Green =
  select, Red = back, plus the d-pad).
- **Handoff:** the wrapper calls `UI_TAKEOVER` before and `UI_RELEASE` after (via `trap`).
- **Host preview:** guard a host build (e.g. `cc -DHOST_PREVIEW ...`) that renders the same frames to
  PNG on your computer, so you can iterate the UI without the device.
- **Reproducible:** same compiler + flags → byte-identical binary; publish its SHA-256 and have the
  build script print it.
- A payload can read the active theme name (via `uci` or the prefs API) and draw a UI that matches it,
  with its own look as a fallback.

## Go (when the tool will grow)
The device's own UI is Go, so Go is proven here and is a strong choice when you want real
strings/slices/`filepath`/`sort`, memory safety, and **free floating point**.

- **Build:** `GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 go build -ldflags="-s -w"
  -trimpath` → roughly **1.5–2 MB** (the Go runtime floor). That's fine on ~251 MB RAM; startup/GC
  overhead is negligible for a UI tool. `GOMIPS=softfloat` is required (no FPU).
- `mmap` `/dev/fb0` and read `/dev/input/event0` with `syscall`/`os` — same model as C, far less
  boilerplate.
- **Trade-off vs C:** ~100× larger binary (≈2 MB vs ≈20 KB), and reproducibility needs the
  `-trimpath`/`-ldflags` flags. Everything else (speed of development, safety, maintainability, floats)
  favours Go for a tool you intend to extend.

## Rust (middle ground)
`mipsel-unknown-linux-musl`, ~300 KB–1 MB, no GC, memory-safe. C-like size with safety, but a steeper
curve and no vendor precedent — choose it only if size *and* safety both matter.

## Choosing
- Small, single-purpose, tiny footprint → **C**.
- Will grow, needs float math / rich text / file handling → **Go**.
- Pure orchestration (API calls, running tools, loot handling) → stay in **bash**.
Don't rewrite a working payload just to change language.
