# Hardware & OS

Verified on firmware **1.1.3** (`/etc/pineapplepager/version`; `DISTRIB_RELEASE 24.10.1`),
kernel 6.6.86.

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
