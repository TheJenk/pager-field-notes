# Hardware & OS

Verified on firmware **1.1.3** (`/etc/pineapplepager/version`; `DISTRIB_RELEASE 24.10.1`),
kernel 6.6.86.

## At a glance
- **Platform:** OpenWrt-based ("Pineapple Pager 24.10.1", target `ramips/mt76x8`, arch
  `mipsel_24kc`), kernel **6.6.86**, taints `no-all busybox`.
- **SoC:** MediaTek **MT7628AN** (ver 1, eco 2), CPU **MIPS 24KEc V5.5** (~385 BogoMIPS).
- **Cross-compile target:** `mipsel-unknown-linux-*` / `GOARCH=mipsle` (24kc). The device's own
  binaries are musl (`/lib/ld-musl-mipsel-sf.so.1`). There are **no compilers or interpreters on the
  device** (no gcc/cc, no python/perl/lua/go) — build on a host and copy over; on-device scripting is
  bash + `awk` + `jq` (+ `sqlite3`).

## SoC and resources
- **MediaTek MT7628AN**, mipsel (MIPS32, little-endian), **no hardware FPU**. Consequences:
  - In C, stay **integer-only**.
  - In Go, build with `GOMIPS=softfloat`.
  - In shell, do floating-point math in `awk` (it has `sin`/`cos`/`sqrt`/`atan2`).
- **RAM:** ~251 MB total, ~95 MB typically available. There's comfortable headroom for a reasonably
  sized helper program.
- **Storage:** eMMC mounted at `/mmc` (ext4, several GB free). The root filesystem is an **overlayfs**:
  a read-only squashfs (`/rom`, the signed image) unioned with a read-write jffs2 overlay
  (`/overlay`). Edits under `/etc`, app dirs, etc. **copy up** to `/overlay`: they persist across
  reboots, never modify the signed image (so there's no runtime signature issue with editing config
  files), and are reverted by deleting the overlay copy. A **firmware update can overwrite them**, so
  keep a copy of anything you add.
- **Flash layout (NOR, `/proc/mtd`):** `u-boot`, `u-boot-env`, `factory`, `firmware` (~64 MB, holding
  `kernel` + the `rootfs` squashfs), `rootfs_data` (jffs2 = the overlay; ~15 MB usable on 2.0.0).
  Bulk user storage is the **eMMC** (internal, soldered; `/dev/mmcblk0p1` → `/mmc`, ext4, ~3.6 GB,
  ~10 MB/s writes). `/root` is `/mmc/root`, so loot lives on the eMMC. `/tmp` is tmpfs (RAM, ~120 MB,
  cleared at reboot).
- **Boot time:** about 90 s to a ready UI on 2.0.0 (up to ~3.5 min on 1.1.x).

See also: [capabilities](capabilities.md) (what's installed/supported) and
[radios & LEDs](radios-and-leds.md).

## The two processes
The device runs two cooperating programs:

- **UI app** — a **Go** binary (large, ~17 MB; stripped but it keeps Go's `pclntab`, so function
  names are recoverable). It draws the screen, dispatches theme targets, and manages the handoff when
  a payload takes over the display. It's managed by the init system (procd); restarting the UI service
  reloads the active theme. It logs under its own process name in the system log.
- **Daemon** — a **C++** daemon handling Wi-Fi / recon / PineAP and inter-process control. It logs
  under its own name too.

Knowing which is which helps when reading `logread`: UI/theme errors come from the UI app, Wi-Fi/recon
errors from the daemon.
