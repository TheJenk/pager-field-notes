# Building native & Go tools

For anything with an on-screen UI, the shell script runs a compiled program that takes over the
framebuffer. There are three realistic choices. Working starter apps for C and Go (verified on
firmware 2.0.0) are in [Pager Payload Skills](../skills/pager-payload-skills.md).

## Why "freestanding" matters
The MT7628 has **no FPU**, and the system libc is musl **soft-float** (`ld-musl-mipsel-sf`). A
static **glibc** binary, or any binary containing hardware floating-point instructions, dies with
**`Illegal instruction`** (SIGILL). This is *verified*: a glibc hard-float static build failed on
the device. So either:
1. build **freestanding, integer-only C** (no libc, raw syscalls), or
2. use **Go with `GOMIPS=softfloat`**, or
3. link against a musl soft-float toolchain that matches the device's ABI.

## Native freestanding-MIPS C (tiny footprint)
Produces a 2–20 KB static binary with zero dependencies and instant start.

- **Toolchain:** a `mipsel` GCC cross-compiler. On macOS:
  `brew install messense/macos-cross-toolchains/mipsel-unknown-linux-gnu`. On Debian/Ubuntu:
  `gcc-mipsel-linux-gnu`.
- **Flags:**
  `-O2 -march=24kc -msoft-float -fno-pic -mno-abicalls -G0 -nostdlib -ffreestanding -static -s -Wl,-e,_start`
- **Prove no FPU instructions slipped in:**
  `objdump -d app | grep -cE '\$f[0-9]|lwc1|swc1|cvt\.|mtc1|mfc1'` must print `0`.
- **No libc:** make raw **o32 syscalls** inline (syscall number in `$2`, args in `$4`–`$6`;
  `$7` is non-zero on error). Stay integer-only, and provide your own `memcpy`/string helpers and a
  small bitmap font. Entry is your own `_start` that calls `main` and then `exit_group`.
- **o32 syscall numbers:** read 4003, write 4004, open 4005, close 4006, getpid 4020, lseek 4019,
  kill 4037, ioctl 4054, nanosleep 4166, poll 4188, getdents64 4219, exit_group 4246,
  clock_gettime 4263.
- **MIPS flag values differ from x86:** `O_NONBLOCK=0x80`, `O_APPEND=0x8`, `O_CREAT=0x100`,
  `O_TRUNC=0x200` (so `O_WRONLY|O_CREAT|O_TRUNC` is `0x301`).
- **Screen:** keep a back buffer, then `lseek(fb, 0, 0)` and do one `write` of 213,120 bytes per
  frame. That's simpler than `mmap`, which takes 6 arguments and passes some on the stack in o32.
  The rotation is `fb[x*222 + (221-y)]` (see [display & input](../device/display-and-input.md)).
- **Pacing:** use `poll()` on the input fd with a ~50 ms timeout as the frame timer. That's ~20 fps
  (the panel's max), wakes immediately on a button, and uses **0 % CPU when idle**. For steady music
  or animation timing, schedule against `clock_gettime` deadlines rather than fixed sleeps.
- **Files without libc:** list a directory with `getdents64` (`d_reclen` u16 at offset 16,
  `d_type` at 18 with 4 = dir and 8 = file, name at 19). Get a file's size with
  `lseek(fd, 0, SEEK_END)`.
- **Startup:** the keypress that launched the payload is often still queued, so drain
  `/dev/input/event0` and ignore input for the first ~150–400 ms.
- **Host simulation:** keep the app logic shared and put only the I/O behind `#ifdef HOST_PREVIEW`.
  Then a host build can take scripted key presses (`./app_host UP GREEN RED`) and write one image
  per step, so you can test layout and logic without a device.
- **Reproducible:** the same compiler and flags give a byte-identical binary. Publish its SHA-256
  and have the build script print it.

## Go (when the tool will grow)
The device's own UI is Go, so Go is proven here. It's a strong choice when you want real
strings/slices/`filepath`/`sort`, `image/draw`, fonts (`golang.org/x/image/font/basicfont`),
memory safety, and **floating point in software**.

- **Build:** `GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 go build -ldflags="-s -w"
  -trimpath`. The result is **~2–6 MB**, which is fine on ~251 MB RAM. `GOMIPS=softfloat` is
  required.
- Open `/dev/fb0` and `WriteAt` the converted frame each tick. Read `/dev/input/event0` in a
  goroutine (blocking 16-byte reads).
- A `-preview` mode that renders the same frames to PNG on your computer makes iteration fast.
- **Trade-off vs C:** a much larger binary, and reproducibility needs the `-trimpath`/`-ldflags`
  flags. Everything else (development speed, safety, maintainability, floats) favours Go for a tool
  you intend to extend.

## Rust (middle ground)
`mipsel-unknown-linux-musl`, ~300 KB–1 MB, no GC, memory-safe. You get C-like size with safety,
but a steeper learning curve and no precedent on this device. Choose it only if size *and* safety
both matter.

## Performance budget — *measured*
- A full-screen native app redrawing at 20 fps with heavy work (an 8-bit console emulator) used
  ~55–70 % CPU.
- Demodulating a 2 MS/s SDR stream (ADS-B) used ~30 % for the decoder alone.
- In bash, every `$(…)` or pipe forks, at ~15–30 ms each. A redraw that forks 40 times takes about a
  second; the same redraw written fork-free took ~90 ms. **Never fork in a per-frame or
  per-keypress path.**

## Choosing
- Small, single-purpose, tiny footprint → **C**.
- Will grow, needs float math, rich text or file handling → **Go**.
- Pure orchestration (API calls, running tools, loot handling) → stay in **bash**.
Don't rewrite a working payload just to change language.
