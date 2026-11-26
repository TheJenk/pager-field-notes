---
name: pager-payload-skills
description: Everything needed to design, write, test and ship payloads for the Hak5 WiFi Pineapple Pager - device internals, screen/framebuffer (rotation, 20 fps), button input, UI_TAKEOVER, DuckyScript commands, the local API, GPS/Bluetooth/Wi-Fi on firmware 1.1.x and 2.0, native C and Go on-screen apps, background watchers, serial peripherals, testing with or without a Pager connected, dependency checks, and repo submission. Use whenever someone wants to build, fix, or understand a Pager payload.
---

# Pager Payload Skills

A single reference file for building **payloads** for the **Hak5 WiFi Pineapple Pager** (themes
are a separate topic). Give it to an AI coding assistant (Claude Code, Cursor, ChatGPT, or
another) or read it yourself. It collects hands-on findings from testing a retail Pager,
including several behaviours that aren't in the official docs.

> **Unofficial, not affiliated with Hak5.** These are observations in our own words, with no Hak5
> firmware, binaries or documentation text. The official docs live at
> `docs.hak5.org/wifi-pineapple-pager`. The Pager is a security-testing tool: only use payloads on
> networks and devices you own or are authorized to test.
>
> **Versions.** Verified on firmware **2.1.0** and **2.0.0** (Nov 2026) and **1.1.3**. When a fact differs between
> versions, this file says which version it applies to. Firmware changes things, so check the
> version first (section 3).

**Using this as a Claude Code skill:** save it as `~/.claude/skills/pager-payload-skills/SKILL.md`.
For other assistants, paste it into the chat or attach it as project knowledge.

---

## 0. Instructions for the AI assistant

You're helping someone build a Pager payload. **Assume you have no access to the Pager.** You
work on the user's computer (or only in chat), and the user installs and runs things on the
device. Everything in this file is designed to work that way. If you *do* have SSH to the Pager,
section 18 adds faster on-device testing, but nothing depends on it.

### Access levels: pick the one that matches, never assume a higher one
| Level | What you can do | How you test |
|---|---|---|
| **A. No device access** (default) | write files on the user's computer, run local tools | mock runs, host previews, `pager-check.sh`, then the user tests on the Pager using your checklist |
| **B. User relays** | as A, plus the user runs commands on the Pager (SSH, or the Virtual Pager web terminal on 2.x) and pastes the output | as A, plus targeted diagnostics you ask for |
| **C. Agent has SSH** | run commands on the Pager yourself | as A, plus on-device runs, screen grabs and logs (section 18) |

### The workflow (definition of done)
1. **Clarify the idea:** what triggers it (menu launch, background watcher, alert event, recon
   target, boot), what it shows, and what it saves. Ask the **firmware version** (*Settings →
   About* on the Pager). If unknown, support both 1.1.x and 2.x (section 3).
2. **Check dependencies** with `pager-doctor.sh` (section 1). Offer the install commands it prints.
   Plain bash payloads need nothing.
3. **Pick the simplest form** (section 5). Bash plus built-in dialogs covers most ideas. Go native
   (C or Go) only for custom full-screen graphics or speed.
4. **Write it** using the patterns and gotchas here instead of guessing. Most were learned the
   hard way.
5. **Test offline** (section 13). This is required:
   - `bash mock-ducky.sh payload.sh` with `MOCK_ANSWERS` for each menu path, plus `MOCK_FW=1.1.3`
     if you support old firmware, and `MOCK_EVENT` for alert/recon payloads. Read the transcript.
   - Native apps: host-preview builds driven by scripted key presses. Look at the PNGs.
   - `bash pager-check.sh <folder>` must report **0 FAIL**. It also packages the payload.
6. **Hand over** the folder, the `dist/` package, the generated `INSTALL-<name>.txt`, and a short
   **on-device test checklist** for the user (section 13.6). Say plainly what you verified
   offline and what only a real Pager can confirm (button feel, GPS fix, radios, timing).
7. **Iterate from the user's report.** Ask for specific things: what the screen showed (a phone
   photo is fine), the payload log, and on 2.x `logread | tail -50` from the Virtual Pager terminal.

### Ground rules
- Never claim something works on the device unless it ran there.
- Warn before anything that could stop the Pager UI, change radios or networking, or modify
  system files. Prefer reversible changes and ship an uninstall note.
- Use placeholders for anything personal (SSIDs, keys, URLs, home coordinates).

---

## 1. Toolkit and dependencies

This file embeds four helper scripts. Save them next to your payload projects; the AI can write
them out from here.

| Script | Section | Does |
|---|---|---|
| `pager-doctor.sh` | 1 | checks and installs build tools; optionally probes a Pager |
| `mock-ducky.sh` | 13 | runs `payload.sh` on your computer with fake Pager commands, API, events |
| `pager-check.sh` | 13 | lints a payload folder for Pager-specific mistakes, then packages it with install steps |
| `fb2png.py` | 7 | turns a raw screen grab into a PNG (no third-party modules) |

What you need to build depends on the kind of payload:

| Payload kind | Needs on your computer |
|---|---|
| bash only (dialogs, tools, API calls) | a text editor. **Nothing to compile.** (Installing uses `scp`/`ssh` or just a browser, section 12.) |
| native C on-screen app | a **mipsel cross-compiler** (plus a host `cc` for previews) |
| Go on-screen app | **Go** only. It cross-compiles from Mac, Linux or Windows with no extra toolchain, which makes it the easiest native option for most people. |
| testing/packaging | `python3` (binary checks in `pager-check.sh`), `shellcheck`, `zip`, `jq` (to test JSON parsing locally) |
| nice-to-have | `ffmpeg` (screen grabs to PNG/MP4), `git` |

Install commands:

| Tool | macOS (Homebrew) | Debian/Ubuntu/WSL | Windows |
|---|---|---|---|
| mipsel cross-gcc | `brew install messense/macos-cross-toolchains/mipsel-unknown-linux-gnu` | `sudo apt install gcc-mipsel-linux-gnu` | use WSL |
| Go | `brew install go` | `sudo apt install golang-go` (or go.dev/dl for a newer one) | installer from go.dev/dl |
| ffmpeg / shellcheck | `brew install ffmpeg shellcheck` | `sudo apt install ffmpeg shellcheck` | WSL, or `winget install ffmpeg` |
| host C compiler | `xcode-select --install` | `sudo apt install build-essential` | WSL |

**The doctor script.** Save it as `pager-doctor.sh` and run `bash pager-doctor.sh`. It reports
what's present and prints the install command for what's missing. `--install` offers to run that
command. Only if SSH to the Pager is set up, `PAGER=172.16.52.1 bash pager-doctor.sh` also
probes the device.

```bash
#!/usr/bin/env bash
# pager-doctor.sh - check (and optionally install) what you need to build Pager payloads.
#   bash pager-doctor.sh            # report only
#   bash pager-doctor.sh --install  # also try to install what's missing (asks first)
#   PAGER=172.16.52.1 bash pager-doctor.sh   # also probe a connected Pager over SSH
set -u
INSTALL=0; [ "${1:-}" = "--install" ] && INSTALL=1
OS=$(uname -s); PM=""
if [ "$OS" = Darwin ]; then command -v brew >/dev/null && PM=brew
elif command -v apt-get >/dev/null; then PM=apt
elif command -v dnf >/dev/null; then PM=dnf
elif command -v pacman >/dev/null; then PM=pacman; fi
grep -qi microsoft /proc/version 2>/dev/null && OS="WSL"

ok(){ printf '  \033[32m✔\033[0m %-26s %s\n' "$1" "$2"; }
no(){ printf '  \033[33m✘\033[0m %-26s %s\n' "$1" "$2"; MISSING+=("$3"); }
MISSING=()
echo "Host: $OS  package manager: ${PM:-none found}"
echo "== Needed for every payload"
command -v bash >/dev/null && ok bash "$(bash --version | head -1 | sed 's/.*version //;s/ .*//')" || no bash "" bash
for t in ssh scp; do command -v $t >/dev/null && ok $t "found" || no $t "copy files to the Pager" openssh; done
echo "== Native C payloads (tiny, fast on-screen tools)"
CC=""; for c in mipsel-unknown-linux-gnu-gcc mipsel-linux-gnu-gcc mipsel-openwrt-linux-musl-gcc; do command -v $c >/dev/null && CC=$c && break; done
[ -n "$CC" ] && ok "MIPS cross-compiler" "$CC" || no "MIPS cross-compiler" "needed for C payloads" mipsel-gcc
command -v cc >/dev/null && ok "host C compiler" "for HOST_PREVIEW builds" || no "host C compiler" "for HOST_PREVIEW builds" cc
echo "== Go payloads (easiest cross-platform option)"
if command -v go >/dev/null; then ok go "$(go version | awk '{print $3}')"; else no go "cross-compiles to the Pager with no extra toolchain" go; fi
echo "== Helpful"
command -v ffmpeg >/dev/null && ok ffmpeg "turn screen grabs into PNG/MP4" || no ffmpeg "turn screen grabs into PNG/MP4" ffmpeg
command -v python3 >/dev/null && ok python3 "helper scripts on your computer" || no python3 "helper scripts on your computer" python3
command -v shellcheck >/dev/null && ok shellcheck "lint payload.sh" || no shellcheck "lint payload.sh" shellcheck
command -v git >/dev/null && ok git "submit payloads" || no git "submit payloads" git
command -v zip >/dev/null && ok zip "package payloads" || no zip "package payloads (tar.gz still works)" zip
command -v jq >/dev/null && ok jq "test JSON parsing locally" || no jq "test JSON parsing locally" jq

pkg(){ # map a need to a package for this package manager
  case "$PM:$1" in
    brew:mipsel-gcc) echo "messense/macos-cross-toolchains/mipsel-unknown-linux-gnu";;
    apt:mipsel-gcc)  echo "gcc-mipsel-linux-gnu";;
    dnf:mipsel-gcc|pacman:mipsel-gcc) echo "";;   # not packaged: use Go, or an OpenWrt SDK
    apt:go) echo golang-go;; dnf:go) echo golang;; *:go) echo go;;
    apt:openssh) echo openssh-client;; *:openssh) echo openssh;;
    brew:cc) echo "";;                             # macOS: xcode-select --install
    apt:cc) echo build-essential;; dnf:cc) echo gcc;; pacman:cc) echo gcc;;
    *) echo "$1";;
  esac
}
if [ ${#MISSING[@]} -gt 0 ]; then
  echo; echo "Missing: ${MISSING[*]}"
  PKGS=(); for m in "${MISSING[@]}"; do p=$(pkg "$m"); [ -n "$p" ] && PKGS+=("$p"); done
  case $PM in
    brew) CMD="brew install ${PKGS[*]}";;
    apt) CMD="sudo apt-get install -y ${PKGS[*]}";;
    dnf) CMD="sudo dnf install -y ${PKGS[*]}";;
    pacman) CMD="sudo pacman -S --needed ${PKGS[*]}";;
    *) CMD="";;
  esac
  [ "$OS" = Darwin ] && [ -z "$PM" ] && echo "Install Homebrew first: https://brew.sh"
  [[ " ${MISSING[*]} " == *" cc "* ]] && [ "$OS" = Darwin ] && echo "Host C compiler: xcode-select --install"
  if [ -n "$CMD" ] && [ ${#PKGS[@]} -gt 0 ]; then
    echo "To install: $CMD"
    if [ $INSTALL = 1 ]; then read -r -p "Run it now? [y/N] " a; [ "$a" = y ] && eval "$CMD"; fi
  fi
  echo "Windows: use WSL (Ubuntu) and run this script there, or install Go for Windows and build Go payloads."
fi

if [ -n "${PAGER:-}" ]; then
  echo; echo "== Pager at $PAGER"
  if ssh -o ConnectTimeout=5 -o BatchMode=yes root@"$PAGER" true 2>/dev/null; then
    ssh root@"$PAGER" 'printf "  firmware %s\n" "$(head -1 /etc/pineapplepager/version)"
      command -v blue5ctl >/dev/null && echo "  bluetooth: blue5 (2.x)" || echo "  bluetooth: BlueZ (1.1.x)"
      uci -q get gpsd.core.enabled | grep -q 0 && echo "  gps: read by pineapd (/api/pineap/gps/get)" || echo "  gps: gpsd (gpspipe)"
      df -h /mmc | awk "NR==2{print \"  eMMC free: \"\$4}"
      ls /root/payloads/user 2>/dev/null | tr "\n" " " | sed "s/^/  payload categories: /"; echo'
  else
    echo "  can't SSH as root@$PAGER (key not set up, wrong address, or USB link down)."
    echo "  Try: ssh root@$PAGER   (password = the one set during Pager setup)"
  fi
fi
```

**No `python`, compiler or package build tools exist on the Pager itself.** Build on your computer,
then copy the result over. On-device scripting is bash 5.2, busybox tools, `awk`, `jq` and `sqlite3`.

---

## 2. The device at a glance

| Item | Value |
|---|---|
| SoC / CPU | MediaTek **MT7628AN**, single-core **MIPS 24KEc** (~580 MHz), **little-endian**, **no FPU** |
| OS | OpenWrt-based (24.10.1), Linux 6.6, musl **soft-float** userland (`ld-musl-mipsel-sf`) |
| RAM | ~251 MB total, ~100 MB typically free |
| Storage | internal **eMMC** at `/mmc` (ext4, ~3.6 GB, ~10 MB/s writes). `/root` → `/mmc/root`. It's soldered eMMC, *not* an SD card. |
| Root FS | read-only squashfs `/rom` + small writable jffs2 overlay (~15 MB) unioned as `/` |
| `/tmp` | tmpfs (RAM, ~120 MB). Use it for live state files; it's cleared at reboot. |
| Display | 480×222 landscape colour LCD (ST7796U over SPI), panel max **20 fps** |
| Input | D-pad (Up/Down/Left/Right), **Green** and **Red** buttons, Power |
| Feedback | RGB LEDs behind the D-pad, LEDs under Green/Red, piezo buzzer, vibration motor |
| Radios | 2.4 GHz built-in (phy0) + **MT7921 tri-band 2.4/5/6 GHz** (phy1), Bluetooth (classic + BLE) |
| GPS | none built in. Plug a USB GPS into the host port. |
| USB | **USB-C = device mode** (Ethernet-over-USB + charging to your computer). **USB-A = host** for GPS, Wi-Fi dongles, serial gadgets, flash drives, SDRs. Both work at the same time. |
| Battery | TI BQ27546 fuel gauge (percent, charging state, battery temperature) |
| Shell | **bash 5.2** (`${x^^}`, associative arrays and `$EPOCHSECONDS` all work), busybox |

Two programs make it a "Pager":
- **`/pineapple/pineapple`**: a large **Go** program. It draws the UI, runs themes and payloads, and
  serves the **local HTTP API** on the unix socket `/tmp/api.sock`. Its service is
  `/etc/init.d/pineapplepager`, and restarting that restarts the UI.
- **`pineapd`**: a C++ daemon for Wi-Fi recon, PineAP, handshakes and (on 2.0) GPS.
Both log to `logread`; filter with `logread | grep -i pineap`.

**Installed and handy:** `bash curl wget jq sqlite3 awk sed grep find timeout setsid flock killall
pgrep logger stty base64 gzip hexdump nc tcpdump iw iwinfo nmap openssl uci ubus opkg fbi
hcxtools`, the GPS tools (`gpspipe` and friends; gpsd is off by default on 2.0), and `i2c*`.
On 2.0 there's also `blue5ctl`. **RTL-SDR tools ship too** (`rtl_sdr`, `rtl_tcp`, `rtl_power`,
`rtl_test`), so a USB RTL-SDR dongle works out of the box.
**Fonts on the device:** `/usr/share/fonts/ttf-dejavu/DejaVuSans.ttf`, and the UI's own typeface
`/pineapple/ui/Steelfish.ttf` plus `/pineapple/ui/MaterialIcons-Regular.ttf`. These are handy if a
native app wants to match the Pager look (rasterize them on your computer into a bitmap font).
**Packages:** `opkg` exists, but the writable root overlay is only ~15 MB. It needs internet on
the Pager, and anything you install that way is undocumented for payload users, so **bundle what
your payload needs inside its folder** instead.
**Not installed:** `python`, `perl`, `lua`, compilers, `pkill`, `nohup`, `socat`, `xxd`, `od`, `bc`,
`inotifywait`, `ffmpeg`, `screen`/`picocom`/`microcom`. Work around them: `pgrep`+`kill` instead
of `pkill`, `setsid` instead of `nohup`, `awk` for maths, `hexdump` for bytes.

**Kernel drivers present:** USB-serial (`cdc_acm cp210x ftdi_sio pl2303 ch341` and more), USB
Ethernet (`r8152 asix ax88179 cdc_ether rndis_host`…), USB Wi-Fi (mt76 family including `mt7921u`),
USB mass storage, and filesystems **ext4, vfat, exfat, ntfs3, cifs**. HID is supported, but there's
**no `xpad`** (Xbox pads only work via usbfs in userspace).

**USB host power is limited.** GPS sticks, Wi-Fi dongles, SDRs and small microcontroller boards
run fine; hungrier devices may brown out. There's one USB-A port (behind an internal hub). Use a
powered hub for several devices.

**Performance budget (measured):** the CPU is modest. A full-screen native app redrawing at
20 fps with real work (an NES emulator) used ~55–70 % CPU. Demodulating ADS-B from an SDR used
~30 % for the decoder alone. In bash, each `$(…)` or pipe costs ~15–30 ms, so a redraw that
forks 40 times takes about a second (section 7). Go's runtime overhead is negligible for UI apps.
An idle native UI that blocks on `poll()` uses 0 % CPU, which is good for battery.

**I2C:** bus `i2c-0` (the battery gauge sits at 0x55), with `i2cdetect`/`i2cget`/`i2cset`. Look,
but don't write to unknown addresses.

**Boot time:** ~90 s on 2.0 (up to ~3.5 min on 1.1.x) until the UI is ready.

---

## 3. Firmware versions: detect, then adapt

```bash
FW=$(head -1 /etc/pineapplepager/version)        # e.g. "2.0.0"
```

| Area | 1.1.x | 2.0.0 (2.1.0 the same unless noted) |
|---|---|---|
| GPS | `gpsd` on the serial GPS; read with `gpspipe -w` | **pineapd reads it natively; gpsd disabled.** `gpspipe` gets nothing. Use `/api/pineap/gps/get` |
| Bluetooth | BlueZ (`bluetoothd`, `bluetoothctl`) | **`blue5` owns the adapter and stops bluetoothd.** `bluetoothctl` **hangs**. Use `blue5ctl` |
| Wi-Fi | you may add your own vif on phy1 | pineapd runs `wlan0mon` + `wlan1mon`. **Sniff those passively; don't retune them.** |
| `GPS_CONFIGURE` | `GPS_CONFIGURE <serial> [speed]` | `GPS_CONFIGURE PINEAP [serial]` or `GPS_CONFIGURE GPSD [serial] [speed]` |
| Theme framework | 0.7 / 0.8 | 0.9 |
| DuckyScript commands | 139 | same set on 2.0.0. 2.1.0 adds `PINEAPPLE_KICK_CLIENT` and `PINEAPPLE_PINEAPD_STOP` / `_RESPAWN` |
| Payload stop (2.1.0) | — | each payload is its own process group; UI "stop" kills it. **`setsid` workers survive**, plain `&` children don't |
| Payload `PATH` (2.1.0) | `/usr/bin:/bin:/usr/sbin:/sbin` | adds `/usr/local/*`, `.`, and **`/mmc/bin` `/mmc/sbin` `/mmc/usr/*`** |
| SSH / audio (2.1.0) | OpenSSH; PWM buzzer | dropbear; buzzer is an ALSA card, `pwmplay file.mp3` plays audio |

Updating to 2.0 **removed installed user payloads** (`/root/payloads/user/*`) and regenerated the
SSH host key. Loot and themes survived. Files you keep elsewhere under `/mmc` survive too.
Updating to **2.1.0 kept** payloads, themes, loot and the host key, but **reset the active theme to
stock** and dropped files added outside `/mmc` (such as links in `/usr/bin`). On 2.1.0 the recon DB
is SQLite WAL (copy `-wal`/`-shm` too), and you must **never run `pineapd --help`**: it starts a
second pineapd that takes over the real one. For UTC timestamps use `date -u`: `date` prints the
configured zone (on 2.0 too, even when it was labelled "UTC"). See [firmware 2.1.0](../firmware/2.1.0.md).

**Support both versions in one payload:**

```bash
# GPS source
if curl -s --unix-socket /tmp/api.sock http://localhost/api/pineap/gps/get | grep -q '"connected"'; then
    GPS=api                                  # 2.x
elif [ -c "$(uci -q get gpsd.core.device)" ]; then
    GPS=gpsd                                 # 1.1.x (by-path symlink resolves to a live tty)
else
    GPS=none
fi
# Bluetooth source
if command -v blue5ctl >/dev/null && blue5ctl ping >/dev/null 2>&1; then BT=blue5; else BT=bluez; fi
# Existing monitor interfaces (2.x: use these, don't make your own)
MONS=$(iw dev | awk '/Interface/{i=$2} /type monitor/{print i}')
```

---

## 4. Payload anatomy and runtime

### Folder and header
A payload is a folder with a `payload.sh` entry point plus anything it needs:

```
/root/payloads/user/<category>/<Payload_Name>/
    payload.sh        # entry point (bash)
    mytool            # optional prebuilt binary
    src/ build.sh     # its source and build script
    README.md
```

Folder names use **no spaces** (underscores or dashes). Existing user categories include
`general`, `reconnaissance`, `exfiltration`, `interception`, `incident_response`,
`remote_access`, `prank`, `games`, `evil_portal` and `examples`. Pick an existing one.

```bash
#!/bin/bash
# Title: My Payload
# Author: YourName
# Description: One line saying what it does.
# Version: 1.0
# Category: general
```

### How a payload actually runs (verified)
- It's launched from the **Payloads** menu and runs as **root**.
- **The UI copies `payload.sh` to `/tmp/payload-<n>.sh` and runs the copy.** So `$0` and
  `dirname "$0"` point at `/tmp`, **not your folder**. Find your own files with
  **`$_PAYLOAD_HOME`** (the payload's real directory), with a fallback:
  ```bash
  DIR="${_PAYLOAD_HOME:-/root/payloads/user/general/My_Payload}"
  [ -x "$DIR/mytool" ] || DIR="$(cd "$(dirname "$0")" && pwd)"
  ```
- `PATH=/usr/bin:/bin:/usr/sbin:/sbin` on 1.1.x/2.0. If you install extra tools elsewhere, add that
  folder to `PATH` yourself. **2.1.0** adds `/mmc/bin`, `/mmc/sbin`, `/mmc/usr/*`, `/usr/local/*` and
  `.` (the current directory).
- `LOG` lines appear on the payload's on-screen log while it runs.
- `/lib/hak5/commands.sh` and `/lib/hak5/pineapple.sh` hold shell helpers (`HAK5_API_GET/POST/PUT`,
  and the dialog constants `DUCKYSCRIPT_USER_CONFIRMED=1`, `DUCKYSCRIPT_USER_DENIED=0`,
  `DUCKYSCRIPT_CANCELLED=2`, `DUCKYSCRIPT_REJECTED=3`, `DUCKYSCRIPT_ERROR=4`).

### The four kinds of payload (by where they live)
| Repo path / device path | Runs when | Extra input |
|---|---|---|
| `library/user/<category>/<Name>` → `/root/payloads/user/…` | user picks it in **Payloads** | — |
| `library/recon/access_point/<Name>` → `/root/payloads/recon/access_point/…` | user selects an **AP in Recon** and runs it on that AP | `$_RECON_SELECTED_AP_*` |
| `library/recon/client/<Name>` → `/root/payloads/recon/client/…` | user selects a **client in Recon** | `$_RECON_SELECTED_CLIENT_*` + the AP's `$_RECON_SELECTED_AP_*` |
| `library/alerts/<event>/<Name>` → `/root/payloads/alerts/<event>/…` | pineapd **event** fires (headless) | `$_ALERT` + `$_ALERT_*` |

**Recon variables** (from Hak5's own examples):
- AP: `_RECON_SELECTED_AP_BSSID`, `_SSID`, `_OUI`, `_HIDDEN`, `_CHANNEL`, `_FREQ`,
  `_ENCRYPTION_TYPE`, `_CLIENT_COUNT`, `_RSSI`, `_PACKETS`, `_TIMESTAMP`, `_MAC_ADDRESS`,
  `_BEACONED_SSID(S)`, `_RESPONDED_SSID(S)`, `_PROBED_SSID(S)`
- Client: `_RECON_SELECTED_CLIENT_MAC_ADDRESS`, `_SSID`, `_BSSID`, `_OUI`, `_CHANNEL`, `_FREQ`,
  `_ENCRYPTION_TYPE`, `_RSSI`, `_PACKETS`, `_TIMESTAMP`, `_PROBED_SSID(S)`

  Each of these continues the prefix (e.g. `$_RECON_SELECTED_AP_SSID`). Values can carry
  stray whitespace, so trim before matching: `b=$(printf %s "$_RECON_SELECTED_AP_BSSID" | tr -d '[:space:]')`.

**Alert payloads** run on **four fixed Wi-Fi events only**. You can't add new event types. One
payload per event is active, chosen in the Alerts dashboard. Each event only fires if the
matching capture feature is on (`pineap/get_config` shows the flags). The payload runs headless
(the screen may be asleep):

| Event folder | Fires when | Variables |
|---|---|---|
| `handshake_captured` | WPA handshake / PMKID saved | `_ALERT_HANDSHAKE_SUMMARY`, `_AP_MAC_ADDRESS`, `_CLIENT_MAC_ADDRESS`, `_TYPE` (eapol/pmkid), `_COMPLETE`, `_CRACKABLE`, `_PCAP_PATH`, `_HASHCAT_PATH` |
| `deauth_flood_detected` | deauth/disassoc flood | `_ALERT_DENIAL_MESSAGE`, `_SOURCE_MAC_ADDRESS`, `_DESTINATION_MAC_ADDRESS`, `_AP_MAC_ADDRESS`, `_CLIENT_MAC_ADDRESS` |
| `pineapple_auth_captured` | enterprise credentials | `_ALERT_AUTH_SUMMARY`, `_TYPE`, `_USERNAME`, `_CHALLENGE_IDENTITY` |
| `pineapple_client_connected` | client joins the Pager's AP | `_ALERT_CLIENT_CONNECTED_SUMMARY`, `_AP_MAC_ADDRESS`, `_CLIENT_MAC_ADDRESS`, `_SSID`, `_SSID_LENGTH` |

As with recon, each name continues the event's prefix
(`_ALERT_HANDSHAKE_PCAP_PATH`, `_ALERT_DENIAL_SOURCE_MAC_ADDRESS`, …). `$_ALERT` holds the event
name. The minimal alert payload is `ALERT "$_ALERT_HANDSHAKE_SUMMARY"`. Client-connected fires
often, so keep a seen-list and a cooldown. There's **no Bluetooth or GPS alert event**; use a
watcher (9.1).

### Other ways code runs
- **Boot payload**: one payload can be set to run once after each boot. It starts after the UI
  finishes booting (the app creates `/tmp/boot_complete` and `/tmp/serverbooted`; wait for them
  if you call the API early). It runs once, so an always-on job must loop inside it. It runs
  headless unless it takes the screen.
- **Background watchers**: a normal payload can detach a process with `setsid` and toggle it off
  next run (section 9.1). This is how you build "alert me when X" for anything that isn't one of
  the four events.
- **From a theme icon? No.** Theme menu items can't launch payloads or run commands on current
  firmware. The hooks exist in the binary but aren't wired up, so don't design around it.
- **From a shell on the Pager** (SSH or the web terminal):
  `cd DIR && setsid bash payload.sh >/tmp/p.log 2>&1 </dev/null &` runs it, and `UI_TAKEOVER`
  works. Dialogs and `LOG` from a shell-launched payload don't always reach the screen, though,
  and killing such a test mid-takeover can wedge the UI (section 14). The Payloads menu is the
  only fully faithful launch, so do the final test from it.

---

## 5. Choosing the form

| You want | Use |
|---|---|
| Menus, prompts, run tools, save loot, notify | **bash + built-in dialogs** (`LIST_PICKER`, `PROMPT`, `ALERT`…) |
| Show a picture or QR | `PROMPT_IMAGE file.png` / `PROMPT_QR "data" "message"` (no code needed) |
| "Alert me when X" in the background | **bash watcher** (section 9.1) + `ALERT` |
| Act on a specific AP/client the user picked in Recon | **recon payload** (section 4) |
| React to a handshake / deauth flood / captured creds / client join | **alert payload** (section 4) |
| Custom full-screen live display, game, HUD, map | **native app** that takes over the screen: |
| …tiny, single-purpose, fastest start | **C, freestanding** (2–20 KB binary) — section 10 |
| …will grow, needs strings/JSON/floats/text easily | **Go** (`GOMIPS=softfloat`, ~2 MB) — section 11 |
| Live data on a custom screen | **split**: bash gathers data → `/tmp` file → native renderer draws (9.3) |

Don't rewrite a working payload just to change language.

---

## 6. Built-in commands (DuckyScript)

All the UPPERCASE commands in `/usr/bin` are symlinks to one multicall binary (`hak5cmd`), which
turns each call into a request to the local API. **Prefer these over hand-rolled versions**,
because they integrate with the UI.

### Dialogs and UI (the app draws these; the payload waits)
| Command | Use | Returns |
|---|---|---|
| `LOG [color] "msg"` | line in the payload log (colors: green yellow red blue) | — |
| `ALERT "msg"` | modal popup | — (**not** written to logread) |
| `PROMPT "msg"` | modal; waits for dismiss | — |
| `ERROR_DIALOG "msg"` | error modal | — |
| `CONFIRMATION_DIALOG "msg"` | yes/no | prints `1` (yes) / `0` (no); exit status = 3 rejected, 4 error |
| `LIST_PICKER "title" opt1 … optN default` | choose from a list (last arg = default) | prints chosen option text |
| `TEXT_PICKER "title" "default"` | on-screen keyboard | text |
| `NUMBER_PICKER` / `IP_PICKER` / `MAC_PICKER "title" default` | typed value pickers | value |
| `START_SPINNER "text"` / `STOP_SPINNER <id>` | busy indicator | start prints an id |
| `WAIT_FOR_INPUT` | wait for any button | prints the button name (run it once to see the names on your firmware) |
| `WAIT_FOR_BUTTON_PRESS <button>` | wait for one specific button | — |
| `PROMPT_QR "data" "message"` | QR code modal | — |
| `PROMPT_IMAGE /path/img.png` | image modal (**PNG or JPEG only**) | — |
| `UI_TAKEOVER` / `UI_RELEASE` | give your program the screen and buttons, then hand them back | — |
| `DISABLE_DISPLAY` / `ENABLE_DISPLAY` | backlight off/on (not a screen takeover) | — |
| `RESTART_UI` | restart the Pager UI | — |

Patterns (straight from how the stock examples do it):
```bash
choice=$(LIST_PICKER "Main Menu" "Scan" "Settings" "Exit" "Scan")
resp=$(CONFIRMATION_DIALOG "Delete loot?") || exit 1
[ "$resp" = "$DUCKYSCRIPT_USER_CONFIRMED" ] && rm -rf "$LOOTDIR"
id=$(START_SPINNER "Working"); do_work; STOP_SPINNER "$id"
```

### Feedback
| Command | Use |
|---|---|
| `RINGTONE [--vibrate] <name or RTTTL>` | play a tune. Built-in names: `alert battery_critical battery_low bonus bootup deskphone error getkey getmachine hack_stealth hak5_the_planet halt health leveldone mute noway pager power_connected power_disconnected pushwall ringring single_beep urgent warning xp yeah`. Or pass an RTTTL string `"Name:d=16,o=5,b=400:c,e,g"`. |
| `ALERT_RINGTONE` | play the user's alert tone |
| `VIBRATE [rtttl or name]` | buzz the motor in a rhythm |
| `DPADLED <red\|green\|blue\|cyan\|yellow\|magenta\|white\|off>` | D-pad LED colour |
| `DPADLED_CONFIG <color>` | set the default D-pad colour |
| `LED <COLOR> [PATTERN]` / `LED <STATE>` | status LED (run `LED --help` for patterns) |
| `DO_A_BARREL_ROLL` | looping D-pad colour chase (stop it with `DPADLED off`) |

### Device info
`BATTERY_PERCENT`, `BATTERY_CHARGING`, `GPS_GET` (prints `lat lon alt speed`), `GPS_LIST`,
`GPS_CONFIGURE`, `FIND_CLIENT_IP <mac>`.

### Wi-Fi / PineAP / recon
`PINEAPPLE_RECON_NEW`, `PINEAPPLE_HOPPING_START/STOP`, `PINEAPPLE_SET_BANDS <iface> 2 5 6`,
`PINEAPPLE_EXAMINE_BSSID <bssid> [secs]`, `PINEAPPLE_EXAMINE_CHANNEL <ch> [secs]`,
`PINEAPPLE_EXAMINE_RESET`, `PINEAPPLE_DEAUTH_CLIENT[_ALL|_CSA]`, `PINEAPPLE_MIMIC_ENABLE/DISABLE`,
the `PINEAPPLE_{DEVICE,NETWORK}_FILTER_{ADD,ADD_FILE,DELETE,CLEAR,LIST,MODE}` and
`PINEAPPLE_SSID_POOL_{ADD,ADD_FILE,DELETE,CLEAR,LIST,START,STOP,COLLECT_START,COLLECT_STOP}`
families, `PINEAPPLE_LOOT_ARCHIVE`.
AP/client: `WIFI_CONNECT <iface> <ssid> <enc> [key]`, `WIFI_DISCONNECT`, `WIFI_CLEAR`, `WIFI_WAIT`,
`WIFI_{MGMT,OPEN,WPA}_AP[_CLEAR|_DISABLE|_HIDE]`, `WIFI_PCAP_START/STOP`.
WiGLE: `WIGLE_START` (prints the CSV path; rows only get logged **with a GPS fix**), `WIGLE_STOP`,
`WIGLE_LOGIN`, `WIGLE_LOGOUT`, `WIGLE_UPLOAD`.

### Network services, storage, config
`DNSSPOOF_*`, `AUTOSSH_*`, `OPENVPN_*`, `WIREGUARD_*`, `SYSTEM_DNS`, `SSH_ADD_KNOWN_HOST`,
`USB_STORAGE`, `USB_WAIT`, `USB_EJECT`, `USB_FREE`,
**`PAYLOAD_SET_CONFIG <payload> <option> <value>` / `PAYLOAD_GET_CONFIG` / `PAYLOAD_DEL_CONFIG`**
(persistent settings that survive reboots; use them instead of your own config files),
`PAGERPORTAL_PAYLOAD_*`, `RESTORE_DEFAULT_THEME`, `DEVELOPER_THEME_RELOAD`, `INSTALL_FIRMWARE`,
`PASSWORD`.

### Command gotchas (verified)
- **`ALERT` queues behind an open `LIST_PICKER`.** It only appears once the menu closes. For
  feedback inside a menu flow, use `PROMPT`.
- **`ALERT` isn't logged.** Add `logger -t MyPayload "msg"` if you want a record in `logread`.
- **`WAIT_FOR_INPUT` and the dialogs need the UI.** During `UI_TAKEOVER`, read
  `/dev/input/event0` yourself.
- **Sound respects the user's mute.** If the Pager's volume preference is 0, `RINGTONE` and
  `ALERT_RINGTONE` are silent. The app also refuses a second `RINGTONE` while one is playing.
- `UI_TAKEOVER`, `UI_RELEASE`, `DISABLE_DISPLAY`, `ENABLE_DISPLAY`, `DO_A_BARREL_ROLL`,
  `USB_STORAGE` and `USB_WAIT` **ignore `--help` and just run.** Don't loop `--help` over every
  command.
- `GPS_LIST` can show a GPS that was unplugged (stale cache, section 8.4).
- `PINEAPPLE_SET_BANDS wlan1mon 2` returned success on 2.0, but hopping still visited 5 GHz.
  Verify that it took effect.

---

## 7. The screen: framebuffer, rotation, 20 fps, takeover

### Facts
- Device **`/dev/fb0`**, **222 × 480**, **RGB565 little-endian**, stride 444 bytes,
  **213,120 bytes per frame**. No padding.
- **The panel is rotated 90°.** The upright view is **landscape 480 × 222**, but memory is laid
  out as 480 columns of 222 pixels, each column flipped. For an upright pixel `(x 0..479, y 0..221)`:
  ```
  index = x * 222 + (221 - y)          // u16 index into the framebuffer
  byte  = index * 2                    // low byte first (little-endian)
  ```
  Draw everything in upright 480×222 space, then map it through this formula. Hak5's own boot
  splash files (`/lib/pager/*.fb`) use the same layout.
- **Panel refresh tops out at 20 fps** (the driver's `fps=20`). Drawing faster wastes CPU and
  battery. Use ~20 fps for animation, or redraw only on input or data change.
- Colour: `rgb565 = (r & 0xF8) << 8 | (g & 0xFC) << 3 | b >> 3`. No red/blue swap is needed.
- Backlight: `/sys/class/backlight/backlight_pwm/brightness` (0–16). The user's brightness and
  timeouts are also in the preferences API (section 8.1).
- Screen dims and sleeps on its timeout even during a takeover, unless the user presses buttons.

### Taking over the screen (the right way)
`UI_TAKEOVER` asks the Pager app to stop drawing and stop handling buttons. `UI_RELEASE` gives them
back. Rules learned on hardware:

1. **Call them from `payload.sh`, not from inside your binary.** If only a child process calls
   `UI_TAKEOVER`, the payload log screen keeps redrawing and the desktop flashes through.
2. **Release on every exit path**, with a `trap`. A payload that dies without `UI_RELEASE` leaves
   the user looking at a frozen frame.
3. **Put `UI_RELEASE` first in cleanup.** Then a slow cleanup step (stopping a scanner, say) can't
   leave the screen stuck.
4. **Draw a first frame immediately**, before any slow setup like opening a serial port or a first
   network fetch. Otherwise the user stares at the old screen.
5. **Drain the launch keypress.** The Green press that started the payload is often still queued
   in the input device. Discard events for the first ~150–400 ms.
6. Old payloads that `SIGSTOP` the UI process or poke the console are obsolete. Use the takeover.

```bash
#!/bin/bash
# Title: My Screen App
# Author: YourName
# Description: Full-screen app. D-pad moves, Green acts, Red exits.
# Version: 1.0
# Category: general
DIR="${_PAYLOAD_HOME:-/root/payloads/user/general/My_Screen_App}"
[ -f "$DIR/app" ] || DIR="$(cd "$(dirname "$0")" && pwd)"
chmod +x "$DIR/app" 2>/dev/null          # browser-upload installs lose the executable bit

cleanup() {
    UI_RELEASE                       # first, always
    # stop helpers here, e.g. kill -TERM -"$HELPER" 2>/dev/null
}
trap cleanup EXIT INT TERM
UI_TAKEOVER
"$DIR/app"
```

### Drawing without a compiler
- **Show an image:** `PROMPT_IMAGE /path/pic.png` (a modal; the app keeps the screen), or
  `fbi` to draw a JPEG/PNG straight to `/dev/fb0` during a takeover.
- **Pure bash:** possible but slow. Pre-build solid-colour byte strings and write them with
  `dd bs=444 seek=<column> conv=notrunc of=/dev/fb0`. One column of the upright image is one
  444-byte row of the framebuffer. That's fine for a coarse meter that updates a few times a
  second, but anything smooth needs a native binary.
- **Bash performance rule:** on this CPU every `$(…)` or pipe costs a fork (~15–30 ms). In a
  per-frame or per-keypress path, avoid `$(…)` entirely. Use `printf -v`, `${var^^}`, `$(<file)`
  (which doesn't fork), `$EPOCHSECONDS`, and functions that set a variable instead of echoing.

### Capturing the screen (debugging, needs SSH; otherwise a phone photo works fine)
From a computer with SSH to the Pager:
```bash
ssh root@172.16.52.1 'cat /dev/fb0' > fb.raw
ffmpeg -f rawvideo -pixel_format rgb565le -video_size 222x480 -i fb.raw -vf transpose=2 shot.png
```
Without ffmpeg, use this pure-Python converter (`python3 fb2png.py fb.raw shot.png`):
```python
#!/usr/bin/env python3
"""fb2png.py - convert a raw Pager framebuffer grab (222x480 RGB565LE) to an upright 480x222 PNG.
usage: python3 fb2png.py fb.raw out.png        (no third-party modules needed)"""
import struct, sys, zlib
W, H = 480, 222
raw = open(sys.argv[1], "rb").read()[: W * H * 2]
rows = []
for y in range(H):
    row = bytearray(b"\x00")                       # PNG filter byte
    for x in range(W):
        i = (x * H + (H - 1 - y)) * 2              # same mapping the device uses
        c = raw[i] | raw[i + 1] << 8
        row += bytes(((c >> 8) & 0xF8, (c >> 3) & 0xFC, (c << 3) & 0xF8))
    rows.append(bytes(row))
def chunk(t, d): return struct.pack(">I", len(d)) + t + d + struct.pack(">I", zlib.crc32(t + d) & 0xFFFFFFFF)
png = b"\x89PNG\r\n\x1a\n" + chunk(b"IHDR", struct.pack(">IIBBBBB", W, H, 8, 2, 0, 0, 0)) \
    + chunk(b"IDAT", zlib.compress(b"".join(rows), 9)) + chunk(b"IEND", b"")
open(sys.argv[2], "wb").write(png)
```
For a short video, loop `cat /dev/fb0` over SSH (USB manages ~5–6 frames/s) and feed the frames
to ffmpeg with the same flags. If the link is slow, `gzip -c /dev/fb0 | base64` on the device and
decode on your computer.

---

## 8. Input, sound, LEDs, sensors, radios

### 8.1 Buttons (evdev)
- Device **`/dev/input/event0`** (name `keys`). Each event is a **16-byte** `struct input_event`:
  bytes 0–7 time, **8–9 type**, **10–11 code**, **12–15 value** (little-endian).
  `type 1` = key; `value 1` = press, `0` = release, `2` = auto-repeat while held.
- **Key codes:**

  | Button | Code |
  |---|---|
  | Up | 103 |
  | Down | 108 |
  | Left | 105 |
  | Right | 106 |
  | **Green** (confirm/select) | **305** |
  | **Red** (back/exit) | **304** |
  | Power | 116 |

  Go by these codes and colours. The device-tree labels (`a-button`/`b-button`, `BTN_SOUTH`/
  `BTN_EAST`) don't map cleanly to colours, so ignore them. Convention: **Green = OK, Red = back,
  and Red on the top screen exits.**
- Leave **Power** unhandled and the system still does its normal screen sleep.
- For **long-press**, time from press to release (~600 ms works well). Fire tap actions on release
  if the same button also has a hold action.
- **Injecting presses** (remote testing): write press + `EV_SYN` + release + `EV_SYN` records into
  `/dev/input/event0`. That needs a tiny native helper, because busybox can't easily write binary.
  After the display dims or a modal opens, injected input can stop updating the UI; restart it
  with `/etc/init.d/pineapplepager restart`.

### 8.2 Sound (buzzer) and vibration
- The buzzer is a **single square-wave tone generator** (one note at a time; no PCM, no ALSA)
  at `/sys/class/leds/buzzer/`:
  - `frequency` = pitch in Hz
  - `brightness` 0–255 = duty (255 ≈ 50 %, the loudest; **0 = silent**)
  - the `volume` file does nothing
- **Writing sysfs directly ignores the user's mute.** Built-in `RINGTONE` honours it, so if you
  play your own music, give the user an on/off setting. **Always write `0` to `brightness` on
  exit**, in your trap too, or a note keeps sounding.
- Time notes with a millisecond clock (`clock_gettime`, o32 syscall 4263, or Go's `time`), not
  with frame counts.
- Vibration: `VIBRATE`, or `RINGTONE --vibrate`.

### 8.3 LEDs
`/sys/class/leds/`: RGB channels per D-pad direction (`up-led-red`, `up-led-green`,
`up-led-blue`, and the same for `down`, `left`, `right`), `a-button-led`, `b-button-led`, and
`buzzer`. Write `brightness`, or use `DPADLED`/`LED`, which go through the app and also work
during a takeover.

### 8.4 Sensors and status
| What | Where |
|---|---|
| Battery % | `/sys/class/power_supply/bq27546-0/capacity` (or `BATTERY_PERCENT`) |
| Charging | `.../bq27546-0/status` → `Charging` / `Discharging` / `Full` / `Not charging` |
| Battery temperature | `.../bq27546-0/temp` (tenths of °C) |
| Radio temperature | `/sys/class/hwmon/hwmon*/temp1_input` (m°C) |
| CPU/SoC temperature | **none readable.** There's no `/sys/class/thermal` zone; the firmware handles overheat itself |
| User prefs | API `pager/preferences/get` (volume, vibrate, brightness, timeouts, `theme_name`…) |
| Time | system clock is **UTC**; busybox `awk` has `systime()` and `strftime()` |

### 8.5 GPS
**2.0+ (pineapd native):**
```bash
curl -s --unix-socket /tmp/api.sock http://localhost/api/pineap/gps/get
# {"lat":..,"lon":..,"alt":..,"speed":..,"fix":3,"connected":true,"hprecision":..,
#  "sats_tracked":..,"sats_strong":..,"signal_quality":..,"readiness":..}
```
~30 ms per call, so polling once a second is fine. A position is usable when `connected` is true
and `fix` ≥ 2. **There's no heading field**; derive course from consecutive positions. `speed` is
in m/s. `GPS_GET` wraps the same data as `lat lon alt speed` but is much slower (~0.7 s).
pineapd auto-detects the baud rate and receiver type. To point it at a device:
`GPS_CONFIGURE PINEAP /dev/serial/by-path/<id>`.

**1.1.x (gpsd):** `gpspipe -w` prints JSON. `TPV` has `mode` (1 none / 2 2D / 3 3D), `lat`, `lon`
and `speed`; `SKY` has `hdop` (always present, unlike the satellite counts). gpsd emits ~1–2 lines
a second even without a fix, so "no lines for ~5 s" reliably means the GPS is gone.
`gpspipe -r` gives raw NMEA. The device is `uci -q get gpsd.core.device` (a by-path symlink).

**Stale-GPS bug (1.1.3, the hotplug script is unchanged in 2.0):** after an unplug, the
`/dev/serial/by-path/*` symlink is left dangling, so `GPS_LIST` keeps listing the device. Test
presence with `[ -c "$(readlink -f /dev/serial/by-path/<id>)" ]`, or rely on data liveness. An
optional, reversible fix: create `/etc/hotplug.d/usb/99-serial-cleanup`:
```bash
[ "$ACTION" = "unbind" ] || [ "$ACTION" = "remove" ] || exit 0
for l in $(find /dev/serial -type l 2>/dev/null); do [ -e "$l" ] || rm -f "$l"; done
```

### 8.6 Bluetooth
**2.0+:** the built-in `blue5` scanner owns the adapter. Read it; don't fight it:
```bash
blue5ctl fetch -n 0 -g 30      # all devices seen in the last 30 s
# fixed-width columns: ADDRESS TYPE PWR AVG MIN MAX TX PKTS FIRST LAST INFO NAME
# INFO = "co 0xXXXX" (company id); NAME = name + tokens svc=XXXX,.. mfr=<hex> sd=<hex>
```
Filters: `-N name`, `-o 0xID` (company), `-r` (skip random addresses), `-d MAC`. Also available:
`blue5ctl status|ping|clear`, the SQLite log `/root/recon/blue5.db` (table `bt_device`, flushed
every 30 s; query it with `sqlite3`), and the API routes `recon/bluetooth/{status,devices,devices/<mac>}`.
Only 16-bit service UUIDs are shown. **Never start `bluetoothd` or call `bluetoothctl` on 2.0.**
bluetoothctl hangs forever, and starting bluetoothd breaks the Pager's own Bluetooth recon.

**1.1.x:** use `bluetoothctl` only:
`bluetoothctl power off; sleep 1; bluetoothctl power on; bluetoothctl --timeout 10 scan on`, then
parse the `Device <MAC> <name>` lines. `btmgmt` and `hciconfig` **hang** inside payloads because
they fight bluetoothd. Match devices by name, service UUID or manufacturer data, since BLE MACs
are randomized.

### 8.7 Wi-Fi
- phy0 = built-in 2.4 GHz. phy1 = MT7921, tri-band 2.4/5/6 GHz. The client (internet) connection
  `wlan0cli` is **2.4 GHz only** and shares phy0.
- **2.0:** pineapd runs `wlan1mon` (phy1, hops 2.4 + 5 GHz) and `wlan0mon` (phy0, **parked**;
  retuning it returns *Resource busy*). The least intrusive way to watch Wi-Fi is passive capture:
  `tcpdump -i wlan1mon -e -n -l type mgt`.
- **1.1.x:** to scan all bands, add a *temporary* interface and remove it on exit:
  `iw phy phy1 interface add scan0 type managed; ip link set scan0 up; iw dev scan0 scan; …;
  iw dev scan0 del`. Bands by `freq`: <2500 = 2.4, <5925 = 5, otherwise 6 GHz.
- Recon data without parsing packets (2.0 API): `recon/wifi/status` (packets/s, totals,
  encryption and band breakdowns), `recon/wifi/devices` (paged), `recon/wifi/devices/<mac>/clients`,
  `recon/wifi/count`.
- Exclude the Pager's own radios from counts: BSSIDs starting `00:13:37` or `02:13:37`.
- WiGLE CSVs land in `/root/loot/wigle/` in WigleWifi-1.6 format
  (`MAC,SSID,AuthMode,FirstSeen,Channel,Frequency,RSSI,CurrentLatitude,CurrentLongitude,AltitudeMeters,AccuracyMeters,…`).
  - Rows are written **only while there's a GPS fix**.
  - pineapd logs each AP **once per boot**, so a parked Pager stops adding rows. That's expected.
  - **Every `WIGLE_START` opens a new log file.** Don't call it if logging is already running.
  - The uci flag `pineapd.@pineapd[0].logwigle` is **config, not live state**. To know whether
    logging is running right now, check whether pineapd holds a `wigle-*.csv` open:
    `ls -l /proc/$(pgrep pineapd | head -1)/fd | grep -o '/[^ ]*wigle-[^ ]*\.csv' | grep -v -- -bt-`
  - If your payload didn't start the log, don't stop it on exit; leave the user's state as it was.
  - 2.0 also writes Bluetooth WiGLE logs (`…-bt-…csv`).

---

### 8.8 The local API (`/tmp/api.sock`)

```bash
api()     { curl -s --unix-socket /tmp/api.sock "http://localhost/api/$1"; }
api_put() { curl -s --unix-socket /tmp/api.sock -X PUT -d "$2" "http://localhost/api/$1"; }
```
Inside a payload you can also source the device's helpers (`. /lib/hak5/commands.sh`) and use
`HAK5_API_GET <route>` / `HAK5_API_POST <route> <json>` / `HAK5_API_PUT`.
The HTTP method matters: a wrong one returns **405**. Useful routes:

| Route | Method | What |
|---|---|---|
| `pager/preferences/get` | GET | volume, vibrate, `lcd_brightness`, `lcd_timeout`, `dim_timeout`, `theme_name`, `theme_path`, ringtones… |
| `pager/preferences/set` | **PUT** partial JSON | e.g. `{"volume":0,"vibrate":false}` → `{"success":true}` |
| `pineap/gps/get` | GET | GPS (2.0) |
| `pineap/get_config` | GET | logging flags (`loghandshake`, `logpcap`, `logwigle`…), loot paths |
| `pineap/hostapd/get_config` | GET | AP state flags |
| `recon/wifi/status`, `recon/wifi/devices`, `recon/wifi/count` | GET | Wi-Fi recon (2.0) |
| `recon/bluetooth/status`, `recon/bluetooth/devices` | GET | Bluetooth recon (2.0) |
| `recon/setup` | GET | WiGLE/GPS/BT settings summary (2.0) |
| `system/serial/list` | GET | the serial list the UI shows (cached; can be stale) |
| `payload/interact/*` | POST | what the DuckyScript dialog commands call |

Parse JSON with **`jq`** (installed). For big lists in a tight loop, `awk` is faster.
Tip: raise `lcd_timeout`/`dim_timeout` through the preferences API while testing, then restore them.

**Launching payloads by API doesn't work on 2.0.** `payloads/{id}/run_payload` is a leftover
route that never receives valid IDs on the Pager.

*Seen in the firmware but not fully verified* (treat as experimental):
`pager/debug/input/emulatekeys` (inject key presses), `payloads/stop_payload`,
`pager/display/screen`, `files/raw/<path>` and `files/zip/<path>` (file access used by the web UI).

---

## 9. Proven patterns

### 9.1 Toggleable background watcher ("alert me when X")
Run it once and it turns on; run it again and it turns off. On the edge, it raises `ALERT` and
writes an audit line.
```bash
#!/bin/bash
# Title: Battery Low Alert
# Author: YourName
# Description: Toggle. Alerts once when battery drops below THRESHOLD, re-arms when charged.
# Version: 1.0
# Category: general
THRESHOLD=20          # percent
INTERVAL=30           # seconds between checks
TAG=BatteryLowAlert
PIDFILE=/tmp/battery_low_alert.pid
CAP=/sys/class/power_supply/bq27546-0/capacity

notify() { ALERT "$1"; logger -t "$TAG" "$1"; }

watch_loop() {
    echo $$ > "$PIDFILE"
    trap 'rm -f "$PIDFILE"; exit 0' TERM INT
    local armed=1 pct
    while :; do
        pct=$(<"$CAP")
        if [ "$armed" = 1 ] && [ "$pct" -lt "$THRESHOLD" ]; then
            notify "Battery low: ${pct}%"; armed=0
        elif [ "$armed" = 0 ] && [ "$pct" -ge $((THRESHOLD + 5)) ]; then
            armed=1                                   # hysteresis: re-arm only after recovery
        fi
        sleep "$INTERVAL"
    done
}

[ "$1" = "__watch" ] && { watch_loop; exit 0; }

if [ -f "$PIDFILE" ] && kill -0 "$(cat "$PIDFILE")" 2>/dev/null; then
    p=$(cat "$PIDFILE")
    kill -TERM -"$p" 2>/dev/null || kill -TERM "$p" 2>/dev/null   # group kill: instant stop
    rm -f "$PIDFILE"
    LOG "Battery Low Alert: OFF"
else
    # the running script is a /tmp copy, so re-exec that copy with bash
    setsid bash "$0" __watch </dev/null >/dev/null 2>&1 &
    LOG "Battery Low Alert: ON (below ${THRESHOLD}%)"
fi
```
Why each piece matters:
- **`setsid`** detaches the watcher so it outlives the payload, and makes it a process-group leader.
- **`kill -TERM -PID`** (note the minus) kills the group, including the `sleep` child, so "off" is
  instant. A plain `kill` waits until the current `sleep` finishes before the trap runs, so with a
  30 s interval "off" lags by up to 30 s.
- **Alert on transitions** with hysteresis, never on every tick.
- `logger -t` lets you prove the alert fired later with `logread | grep BatteryLowAlert`.
- The same skeleton works for USB drive inserted (`/proc/mounts`), new loot
  (`find /root/loot -type f | wc -l`), SSH logins (`logread` "Accepted"), uplink down (ping the
  gateway), storage full (`df`), GPS fix lost (section 8.5), and a geofence (awk equirectangular
  distance).

### 9.2 Make `pgrep -f` safe
busybox has no `pkill`. `pgrep -f "pattern"` also matches **the shell running the command**,
because the pattern is in its own argv. Run over SSH, that kills your session (exit 255). Also,
`pgrep` patterns are regexes, so `+`, `.` and `[` in paths need escaping. Prefer PID files and
`$!`. If you must match:
```bash
for p in $(pgrep -f "[t]cpdump -i wlan1mon"); do [ "$p" != "$$" ] && kill "$p"; done
```
(The `[t]` trick keeps the pattern from matching its own command line.)

### 9.3 Live display: slow data in bash, fast drawing in a binary
- **Producer** (bash, backgrounded with `setsid`): runs the slow work (`curl` the API, `tcpdump`,
  `iw`, GPS), parses it with `awk`, and writes a small state file **atomically**:
  `printf '%s\n' "$data" > /tmp/app.dat.tmp && mv /tmp/app.dat.tmp /tmp/app.dat`.
  Include a generation counter so the reader only reparses on change.
- **Renderer** (C/Go): re-reads `/tmp/app.dat` a few times a second and keeps drawing at up to
  20 fps from its last copy. The UI stays smooth even when data arrives every 1–3 s.
- Keep slow fetches **out of the keypress path**. Redraw from cached data on a key; refresh data
  on a timer.
- Stop producers with a process-group kill in the `payload.sh` trap, *after* `UI_RELEASE`.

### 9.4 Loot
Save output under `/root/loot/<PayloadName>/` (it's on the eMMC, survives updates, and shows up in
the Pager's loot views). Use timestamped files, e.g. `$(date -u +%Y%m%dT%H%M%SZ)`. Delete empty
sessions on exit.

### 9.5 Settings that persist
`PAYLOAD_SET_CONFIG mypayload key value` / `PAYLOAD_GET_CONFIG mypayload key`, or a small file
in your payload folder. `/tmp` is wiped at reboot, and `/root/payloads/user` was wiped by the 2.0
update. Data you need to keep across firmware updates can live under `/mmc/<yourname>/`.

### 9.6 Structuring a bigger app
- **One main loop as a state machine over screens.** Keep `screen=menu|detail|settings`; each
  pass draws the current screen and handles one key. Don't nest blocking loops per screen.
- **Derive timers from the clock; don't tick them.** A stopwatch is `now - start`
  (`$EPOCHSECONDS` in bash, `clock_gettime` in C). It keeps running correctly across screens and
  never drifts.
- **Background work** (scans, GPS polling, downloads) goes in `setsid` helpers that write `/tmp`
  files the loop reads (9.3).
- **Exit-code relaunch:** to switch modes that need a fresh start (e.g. a different data source),
  have the binary exit with a special code and let `payload.sh` loop:
  ```bash
  while :; do "$DIR/app" --source "$SRC"; rc=$?
      [ $rc -eq 10 ] && { SRC=$([ "$SRC" = radio ] && echo net || echo radio); continue; }
      break
  done
  ```
  Save UI state (selection, zoom) to a `/tmp` file so the relaunch picks it up.
- **Match the user's theme (optional):** read `theme_name` from `pager/preferences/get` and pick
  colours accordingly, with your own look as the fallback.

### 9.7 Internet access from a payload
- The Pager only has internet through **client-mode Wi-Fi** (`wlan0cli`, **2.4 GHz only**, shares
  the built-in radio). The USB-C link to your computer doesn't route internet by default.
- Client Wi-Fi can be weak and erratic. Use short `curl` timeouts (`--max-time 10`), retries and a
  cached last result, and **never block the UI on the network**.
- Public APIs rate-limit (some return an *empty success* when throttled instead of an error).
  Treat "empty" as a failure, back off, and offer a fallback source.
- Show a clear "no internet" state rather than a blank screen.

### 9.8 Editing system files (rarely needed)
`/` is an overlay. Edits copy up to `/overlay`, persist across reboots, never touch the signed
image, and can be undone by deleting the overlay copy. A firmware update may revert them. Prefer
**adding** files (like the hotplug script above) over editing stock ones, and keep a backup.

---

## 10. Native C (freestanding, no libc)

**Why freestanding:** the CPU has **no FPU** and the system libc is musl soft-float. A static
**glibc** binary, or anything containing hardware floating-point instructions, dies with
**`Illegal instruction`**. Freestanding C (raw syscalls, integer-only maths) gives a 2–20 KB
binary with zero dependencies that starts instantly.

**Build:**
```bash
mipsel-unknown-linux-gnu-gcc -O2 -march=24kc -msoft-float -fno-pic -mno-abicalls -G0 \
    -nostdlib -ffreestanding -static -s -Wl,-e,_start -o app app.c
# (Debian/Ubuntu compiler name: mipsel-linux-gnu-gcc)
# prove no FPU instructions slipped in - this must print 0:
mipsel-unknown-linux-gnu-objdump -d app | grep -cE '\$f[0-9]|lwc1|swc1|cvt\.|mtc1|mfc1'
# host simulation: same drawing + input code, scripted presses, one frame per step:
cc -DHOST_PREVIEW -o app_host app.c && ./app_host UP UP GREEN RIGHT
#   -> frame_00.ppm (start) ... frame_04.ppm; view them, or: ffmpeg -i frame_04.ppm f4.png
```

**Rules:**
- **No floats or doubles** anywhere (also check helpers and constants). Use fixed point
  (e.g. ×1000, or 16.16), integer square roots, and lookup tables for sin/cos.
- No libc means you write `memcpy`, `strlen`, and int-to-string yourself, and bring a bitmap font.
  A public-domain 5×7 font is ~500 bytes.
- **MIPS constants differ from x86:** `O_NONBLOCK=0x80`, `O_CREAT=0x100`, `O_TRUNC=0x200`,
  `O_APPEND=0x8` (so `O_WRONLY|O_CREAT|O_TRUNC = 0x301`).
- **o32 syscall numbers:** read 4003, write 4004, open 4005, close 4006, lseek 4019, ioctl 4054,
  kill 4037, getpid 4020, nanosleep 4166, poll 4188, getdents64 4219, exit_group 4246,
  clock_gettime 4263. The return convention: register `$7` (a3) is non-zero on error.
- Present a frame with a back buffer: `lseek(fb,0,0)` then one `write` of 213,120 bytes. That's
  simpler than `mmap` (6-argument syscalls on o32 pass args on the stack) and fast enough.
- Directory listing: `getdents64` (`d_reclen` u16 at offset 16, `d_type` at 18 with 4 = dir and
  8 = file, name at 19). File size: `lseek(fd, 0, SEEK_END=2)`.
- Use `poll` on the input fd with a timeout as your frame timer. That way the app sleeps at 0 % CPU
  when idle and wakes immediately on a button.
- **Reproducible builds:** the same compiler and flags give a byte-identical binary. Have
  `build.sh` print `sha256sum app` and put the hash in the README.

**Keep all game/app logic in shared code** and put only the I/O (`present`, `read_key`,
`sleep_ms`, `init`) behind `#ifdef HOST_PREVIEW`. Then the host simulation exercises exactly the
logic the Pager will run. When you add features, extend the scripted-key test and look at the
frames for every screen and state (empty lists, long text, errors).

**Complete starter app.** Verified on a Pager running 2.0.0: draws upright with correct colours,
and the binary is 2.3 KB. The host simulation was checked against expected positions. The D-pad
moves a box, Green changes its colour, Red exits:
```c
/* app.c - minimal full-screen Pager app: freestanding MIPS, no libc, integer-only.
 * D-pad moves a box, Green cycles its colour, Red exits.
 * Device:  see build line in pagerskills.md
 * Preview: cc -DHOST_PREVIEW -o app_host app.c && ./app_host UP UP GREEN RIGHT
 *          (simulates those presses; writes frame_00.ppm, frame_01.ppm ... one per step) */

#define W 480              /* logical (upright) width  */
#define H 222              /* logical (upright) height */
#define NPIX (W * H)
#define FRAME_BYTES (NPIX * 2)

typedef unsigned short u16;
static u16 fb[NPIX];       /* back buffer in framebuffer (rotated) layout */

/* landscape (x,y) -> framebuffer index. The panel is rotated 90 degrees. */
#define FBI(x, y) ((x) * H + (H - 1 - (y)))
#define RGB(r, g, b) (u16)((((r) & 0xF8) << 8) | (((g) & 0xFC) << 3) | ((b) >> 3))

static void px(int x, int y, u16 c) { if ((unsigned)x < W && (unsigned)y < H) fb[FBI(x, y)] = c; }
static void rect(int x, int y, int w, int h, u16 c) {
    for (int j = y; j < y + h; j++) for (int i = x; i < x + w; i++) px(i, j, c);
}
static void clear(u16 c) { for (int i = 0; i < NPIX; i++) fb[i] = c; }

/* input codes (evdev, /dev/input/event0) */
enum { K_UP = 103, K_DOWN = 108, K_LEFT = 105, K_RIGHT = 106, K_POWER = 116, K_RED = 304, K_GREEN = 305 };

#ifdef HOST_PREVIEW
/* ---------- host build: render one frame to a PPM so you can see it without a Pager ---------- */
#include <stdio.h>
#include <string.h>
static int frame_no;
static void present(void) {               /* each frame -> frame_NN.ppm (upright 480x222) */
    char name[32]; snprintf(name, sizeof name, "frame_%02d.ppm", frame_no++);
    FILE *f = fopen(name, "wb"); fprintf(f, "P6\n%d %d\n255\n", W, H);
    for (int y = 0; y < H; y++) for (int x = 0; x < W; x++) {
        u16 c = fb[FBI(x, y)];
        fputc((c >> 8) & 0xF8, f); fputc((c >> 3) & 0xFC, f); fputc((c << 3) & 0xF8, f);
    }
    fclose(f);
}
static char **keyv; static int keyc, keyi, drained;
static int read_key(void) {               /* scripted presses from the command line */
    static const char *n[] = { "UP", "DOWN", "LEFT", "RIGHT", "POWER", "RED", "GREEN" };
    static const int c[] = { K_UP, K_DOWN, K_LEFT, K_RIGHT, K_POWER, K_RED, K_GREEN };
    if (!drained) return 0;               /* the startup drain sees an empty queue */
    if (keyi >= keyc) return K_RED;       /* script finished: exit */
    const char *k = keyv[keyi++];
    for (int i = 0; i < 7; i++) if (!strcmp(k, n[i])) return c[i];
    return 0;
}
static int sleeps;                         /* 1st sleep = startup drain; keys flow after it */
static void sleep_ms(int ms) { (void)ms; if (++sleeps >= 2) drained = 1; }
static int init(void) { return 0; }
static void fin(void) {}
#else
/* ---------- device build: raw MIPS o32 syscalls ---------- */
#define SYS_read 4003
#define SYS_write 4004
#define SYS_open 4005
#define SYS_close 4006
#define SYS_lseek 4019
#define SYS_nanosleep 4166
#define SYS_poll 4188
#define O_RDONLY 0
#define O_RDWR 2
#define O_NONBLOCK 0x80          /* MIPS value, not the x86 one */
static long sys3(long n, long a, long b, long c) {
    register long r2 asm("$2") = n, r4 asm("$4") = a, r5 asm("$5") = b, r6 asm("$6") = c, r7 asm("$7") = 0;
    asm volatile("syscall" : "+r"(r2), "+r"(r7) : "r"(r4), "r"(r5), "r"(r6)
                 : "$1", "$3", "$8", "$9", "$10", "$11", "$12", "$13", "$14", "$15", "$24", "$25", "memory");
    return r7 ? -r2 : r2;
}
/* entry point: argc/argv from the stack, exit_group(main()) */
asm(".text\n.globl _start\n.ent _start\n_start:\n lw $4,0($sp)\n addiu $5,$sp,4\n jal main\n nop\n"
    " move $4,$2\n li $2,4246\n syscall\n nop\n.end _start\n");

static int fbfd = -1, evfd = -1;
static int init(void) {
    fbfd = (int)sys3(SYS_open, (long)"/dev/fb0", O_RDWR, 0);
    evfd = (int)sys3(SYS_open, (long)"/dev/input/event0", O_RDONLY | O_NONBLOCK, 0);
    return (fbfd < 0 || evfd < 0) ? -1 : 0;
}
static void present(void) { sys3(SYS_lseek, fbfd, 0, 0); sys3(SYS_write, fbfd, (long)fb, FRAME_BYTES); }
struct pfd { int fd; short ev, rev; };
static void sleep_ms(int ms) {           /* wait up to ms, wake early on a button */
    struct pfd p = { evfd, 1, 0 };
    sys3(SYS_poll, (long)&p, 1, ms);
}
/* returns a key code on press (value 1) or auto-repeat (value 2), 0 if none pending */
static int read_key(void) {
    unsigned char e[16];                 /* struct input_event is 16 bytes on this 32-bit kernel */
    while (sys3(SYS_read, evfd, (long)e, 16) == 16) {
        int type = e[8] | e[9] << 8, code = e[10] | e[11] << 8, val = e[12];
        if (type == 1 && (val == 1 || val == 2)) return code;
    }
    return 0;
}
static void fin(void) { sys3(SYS_close, fbfd, 0, 0); sys3(SYS_close, evfd, 0, 0); }
#endif

int main(int argc, char **argv) {
#ifdef HOST_PREVIEW
    keyv = argv + 1; keyc = argc - 1;
#else
    (void)argc; (void)argv;
#endif
    if (init() < 0) return 1;
    /* the key that launched us may still be queued: drain it */
    sleep_ms(150); while (read_key()) {}
    static const u16 colours[] = { 0xF800, 0x07E0, 0x001F, 0xFFE0, 0xF81F, 0x07FF };
    int bx = 220, by = 90, ci = 0, frames = 0;
    for (;;) {
        int k = read_key();
        if (k == K_RED) break;
        if (k == K_UP)    by -= 8;
        if (k == K_DOWN)  by += 8;
        if (k == K_LEFT)  bx -= 8;
        if (k == K_RIGHT) bx += 8;
        if (k == K_GREEN) ci = (ci + 1) % 6;
        if (bx < 0) bx = 0; if (bx > W - 40) bx = W - 40;
        if (by < 0) by = 0; if (by > H - 40) by = H - 40;

        clear(RGB(16, 16, 24));
        rect(0, 0, W, 20, RGB(40, 40, 64));          /* top bar: proves "up" is up    */
        rect(0, 0, 20, 20, 0xF800);                  /* red square = top-left corner  */
        rect(W - 20, H - 20, 20, 20, 0x07E0);        /* green square = bottom-right   */
        rect(bx, by, 40, 40, colours[ci]);
        present();
        if (++frames > 20 * 60 * 10) break;          /* safety: quit after ~10 min */
        sleep_ms(50);                                /* ~20 fps = the panel's max */
    }
    fin();
    return 0;
}
```

---

## 11. Go

The Pager's own UI is written in Go, so Go is proven on this chip. It gives you real strings,
JSON, maps, `image/draw`, fonts, and **floats** (done in software).

```bash
GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 \
  go build -trimpath -ldflags="-s -w" -o app .
```
- **`GOMIPS=softfloat` is mandatory** (no FPU), and so is `CGO_ENABLED=0`.
- Binaries are ~1.5–6 MB. That's fine for RAM and storage, but big to ship in a repo.
- Keep a preview mode: `go run . -preview sim -keys GREEN,GREEN,UP` writes `sim_00.png …
  sim_03.png`, running the same `handle()` and `render()` code the device uses. Those PNGs are
  exactly what the Pager will show.
- Use `fb.WriteAt(buf, 0)` each frame, and read input in a goroutine (blocking reads).

**Complete starter app** (verified on a Pager running 2.0.0: renders text upright; 2.3 MB binary).
Setup: `go mod init myapp && go get golang.org/x/image`.
```go
// Minimal Pager app in Go: draws with image/draw + a bitmap font, reads buttons, Red exits.
// Device:  GOOS=linux GOARCH=mipsle GOMIPS=softfloat CGO_ENABLED=0 go build -trimpath -ldflags="-s -w" -o app .
// Preview: go run . -preview out -keys GREEN,GREEN,UP   (writes out_00.png, out_01.png ... per step)
package main

import (
	"encoding/binary"
	"flag"
	"fmt"
	"image"
	"image/color"
	"image/draw"
	"image/png"
	"os"
	"strings"
	"time"

	"golang.org/x/image/font"
	"golang.org/x/image/font/basicfont"
	"golang.org/x/image/math/fixed"
)

const W, H = 480, 222 // upright landscape size

const (
	kUp, kDown, kLeft, kRight, kPower, kRed, kGreen = 103, 108, 105, 106, 116, 304, 305
)

// present converts the upright RGBA image to the Pager's rotated RGB565 framebuffer.
func present(img *image.RGBA, fb *os.File, buf []byte) {
	for y := 0; y < H; y++ {
		for x := 0; x < W; x++ {
			o := img.PixOffset(x, y)
			r, g, b := uint16(img.Pix[o]), uint16(img.Pix[o+1]), uint16(img.Pix[o+2])
			c := (r&0xF8)<<8 | (g&0xFC)<<3 | b>>3
			i := (x*H + (H - 1 - y)) * 2 // landscape (x,y) -> fb byte offset
			buf[i], buf[i+1] = byte(c), byte(c>>8)
		}
	}
	fb.WriteAt(buf, 0)
}

func text(img *image.RGBA, x, y int, s string, c color.Color) {
	d := &font.Drawer{Dst: img, Src: image.NewUniform(c), Face: basicfont.Face7x13,
		Dot: fixed.P(x, y)}
	d.DrawString(s)
}

func render(img *image.RGBA, presses int, last string) {
	draw.Draw(img, img.Bounds(), image.NewUniform(color.RGBA{16, 16, 24, 255}), image.Point{}, draw.Src)
	draw.Draw(img, image.Rect(0, 0, W, 20), image.NewUniform(color.RGBA{40, 40, 64, 255}), image.Point{}, draw.Src)
	text(img, 6, 14, "HELLO PAGER", color.White)
	text(img, 6, 60, "presses: "+itoa(presses), color.RGBA{0, 255, 120, 255})
	text(img, 6, 80, "last key: "+last, color.RGBA{255, 200, 0, 255})
	text(img, 6, 210, "GREEN = count   RED = exit", color.RGBA{160, 160, 160, 255})
}

func itoa(n int) string {
	if n == 0 {
		return "0"
	}
	s := ""
	for ; n > 0; n /= 10 {
		s = string(rune('0'+n%10)) + s
	}
	return s
}

// app state and input handling, shared by the device loop and the preview
type app struct {
	presses int
	last    string
	quit    bool
}

var keyNames = map[uint16]string{kUp: "UP", kDown: "DOWN", kLeft: "LEFT", kRight: "RIGHT", kGreen: "GREEN", kRed: "RED"}

func (a *app) handle(k uint16) {
	if k == kRed {
		a.quit = true
		return
	}
	if n, ok := keyNames[k]; ok {
		a.last = n
	}
	if k == kGreen {
		a.presses++
	}
}

func savePNG(img *image.RGBA, name string) {
	f, _ := os.Create(name)
	png.Encode(f, img)
	f.Close()
}

func main() {
	preview := flag.String("preview", "", "render frames to <prefix>_NN.png on this computer and exit")
	keyList := flag.String("keys", "", "comma-separated presses to simulate with -preview, e.g. GREEN,UP,RED")
	flag.Parse()
	img := image.NewRGBA(image.Rect(0, 0, W, H))
	a := &app{last: "-"}

	if *preview != "" {
		byName := map[string]uint16{}
		for c, n := range keyNames {
			byName[n] = c
		}
		render(img, a.presses, a.last)
		savePNG(img, fmt.Sprintf("%s_00.png", *preview))
		for i, n := range strings.Split(*keyList, ",") {
			if c, ok := byName[strings.ToUpper(strings.TrimSpace(n))]; ok {
				a.handle(c)
			}
			if a.quit {
				break
			}
			render(img, a.presses, a.last)
			savePNG(img, fmt.Sprintf("%s_%02d.png", *preview, i+1))
		}
		return
	}

	fb, err := os.OpenFile("/dev/fb0", os.O_RDWR, 0)
	if err != nil {
		os.Exit(1)
	}
	defer fb.Close()
	ev, err := os.Open("/dev/input/event0")
	if err != nil {
		os.Exit(1)
	}
	defer ev.Close()

	keys := make(chan uint16, 16)
	go func() { // blocking reader; 16-byte input_event: type@8 code@10 value@12
		b := make([]byte, 16)
		for {
			if _, err := ev.Read(b); err != nil {
				close(keys)
				return
			}
			typ, code := binary.LittleEndian.Uint16(b[8:]), binary.LittleEndian.Uint16(b[10:])
			val := int32(binary.LittleEndian.Uint32(b[12:]))
			if typ == 1 && (val == 1 || val == 2) {
				keys <- code
			}
		}
	}()

	buf := make([]byte, W*H*2)
	ignoreUntil := time.Now().Add(200 * time.Millisecond) // drop the keypress that launched us
	tick := time.NewTicker(50 * time.Millisecond)         // 20 fps max
	deadline := time.Now().Add(10 * time.Minute)
	for time.Now().Before(deadline) {
		select {
		case k, ok := <-keys:
			if !ok {
				return
			}
			if time.Now().Before(ignoreUntil) {
				continue
			}
			a.handle(k)
			if a.quit {
				return
			}
		case <-tick.C:
			render(img, a.presses, a.last)
			present(img, fb, buf)
		}
	}
}
```

(Rust also works: target `mipsel-unknown-linux-musl`, soft-float, ~300 KB–1 MB. There's less
precedent on this device, though.)

---

## 12. Installing on the Pager (done by the user)

Run `bash pager-check.sh My_Payload <category>` first. It writes `dist/My_Payload.tar.gz`, a
`.zip`, and `dist/INSTALL-My_Payload.txt` with the exact steps below filled in. Give the user
that file.

Connect the Pager's **USB-C** to the computer. It appears as a USB network adapter, and the
Pager is at **`172.16.52.1`**. The login is `root` with the password set during Pager setup.

**Option A: terminal (macOS/Linux/WSL, or Windows PowerShell with OpenSSH):**
```bash
scp My_Payload.tar.gz root@172.16.52.1:/tmp/          # add -O if scp complains about sftp
ssh root@172.16.52.1 'mkdir -p /root/payloads/user/general && tar -xzf /tmp/My_Payload.tar.gz -C /root/payloads/user/general && chown -R root:root /root/payloads/user/general/My_Payload && chmod +x /root/payloads/user/general/My_Payload/* && rm /tmp/My_Payload.tar.gz'
```
(Verified on a real Pager: files land executable and the binary runs.)

**Option B: browser only, no terminal (firmware 2.x).** Open **`http://172.16.52.1:1471`**
(the Virtual Pager) and log in. Its **Files** page can create folders and **upload files** (drag
and drop into an open folder). Create `/root/payloads/user/general/My_Payload/` and upload every
file from the payload folder, not the archive. Uploaded files may not be executable, which is why
payloads should `chmod +x` their own binaries at startup (the templates do). The Virtual Pager
also has a **web terminal**: that's how a user without SSH can run diagnostic commands for you.

**Option C: Payload Portal** (`PAGERPORTAL_PAYLOAD_INSTALL <key>`) installs payloads that are
already published in the Hak5 repo. The download has a ~60 s limit, so keep payloads small.

Then on the Pager: **Payloads → user → general → My Payload**. Recon payloads appear when you run
a payload on a selected AP or client in Recon; alert payloads are enabled from the Alerts
dashboard.

**Uninstall:** delete the folder (Files page, or `rm -rf /root/payloads/user/general/My_Payload`).
If the payload started a background watcher, run it once more to toggle it off first.

---

## 13. Building and testing with no Pager connected

This is the normal way to work. Do all of it before handing anything to the user.

### 13.1 Run payload.sh with fake Pager commands
Save `mock-ducky.sh`, then run `bash mock-ducky.sh path/to/payload.sh`. Feed menu choices in
order with `MOCK_ANSWERS`, then read the printed transcript (also saved to
`$TMPDIR/pager-mock/transcript.log`):
```bash
MOCK_ANSWERS="Configure IP|1.1.1.1|Ping|Exit|1" bash mock-ducky.sh My_Payload/payload.sh
MOCK_FW=1.1.3 bash mock-ducky.sh My_Payload/payload.sh          # old-firmware code path
MOCK_EVENT=handshake bash mock-ducky.sh Handshake_Alert/payload.sh  # alert payload
MOCK_EVENT=recon_client bash mock-ducky.sh Client_Info/payload.sh   # recon payload
MOCK_API_DIR=./fixtures bash mock-ducky.sh …   # serve your own JSON for API routes
```
Test **every menu path**, cancel/"no" answers, empty data, and the exit path. If a menu loops
forever when answers run out, the mock aborts after `MOCK_MAX_PROMPTS`.
```bash
#!/usr/bin/env bash
# mock-ducky.sh - run a Pager payload.sh on your computer with fake Pager commands.
#
#   bash mock-ducky.sh path/to/payload.sh [args]
#
# Knobs (environment variables):
#   MOCK_ANSWERS="Scan|Settings|1"   answers for pickers/dialogs, used in order ('|' separated);
#                                    when they run out, pickers return their default, confirm says yes
#   MOCK_FW=2.0.0 | 1.1.3            which firmware to imitate (GPS/Bluetooth behaviour)
#   MOCK_EVENT=handshake|deauth|auth|client|recon_ap|recon_client   preset $_ALERT_* / $_RECON_* vars
#   MOCK_BATTERY=80 MOCK_CHARGING=0  MOCK_GPS="lat lon alt speed" MOCK_FIX=3
#   MOCK_MAX_PROMPTS=100             abort after this many dialogs (catches menus that loop forever)
#   MOCK_API_DIR=dir                 serve <route with / replaced by _>.json for local-API calls,
#                                    e.g. dir/pineap_gps_get.json  (built-in samples otherwise)
# Everything the payload "shows" is printed and appended to $MOCK_DIR/transcript.log.
# Nothing touches real hardware. UI_TAKEOVER apps: put a HOST_PREVIEW build where the payload
# expects its binary (or skip it) - a MIPS binary can't run on your computer.

MOCK_DIR="${MOCK_DIR:-${TMPDIR:-/tmp}/pager-mock}"; mkdir -p "$MOCK_DIR"
MOCK_FW="${MOCK_FW:-2.0.0}"
_T="$MOCK_DIR/transcript.log"; _Q="$MOCK_DIR/answers"
_m(){ printf '\033[36m[%s]\033[0m %s\n' "$1" "${*:2}" >&2; printf '[%s] %s\n' "$1" "${*:2}" >>"$_T"; }
_next(){ # pop the next queued answer (file-based, so it works inside $(...) subshells)
  local n; n=$(( $(cat "$MOCK_DIR/count" 2>/dev/null || echo 0) + 1 )); echo $n >"$MOCK_DIR/count"
  if [ $n -gt "${MOCK_MAX_PROMPTS:-100}" ]; then
    _m ABORT "more than ${MOCK_MAX_PROMPTS:-100} dialogs - is a menu looping forever?"; kill -TERM $$; fi
  [ -s "$_Q" ] || return 1
  REPLY=$(head -n1 "$_Q"); tail -n +2 "$_Q" >"$_Q.t"; mv "$_Q.t" "$_Q"; return 0; }

LOG(){ _m LOG "$*"; }
ALERT(){ _m ALERT "$*"; }
PROMPT(){ _m PROMPT "$*"; }
ERROR_DIALOG(){ _m ERROR "$*"; }
CONFIRMATION_DIALOG(){ _m CONFIRM "$*"; _next && echo "$REPLY" || echo 1; }
LIST_PICKER(){ _m LIST_PICKER "$*"; _next && echo "$REPLY" || echo "${!#}"; }
TEXT_PICKER(){ _m TEXT_PICKER "$*"; _next && echo "$REPLY" || echo "$2"; }
NUMBER_PICKER(){ _m NUMBER_PICKER "$*"; _next && echo "$REPLY" || echo "$2"; }
IP_PICKER(){ _m IP_PICKER "$*"; _next && echo "$REPLY" || echo "$2"; }
MAC_PICKER(){ _m MAC_PICKER "$*"; _next && echo "$REPLY" || echo "$2"; }
WAIT_FOR_INPUT(){ _m WAIT_FOR_INPUT; _next && echo "$REPLY" || echo A; }
WAIT_FOR_BUTTON_PRESS(){ _m WAIT_FOR_BUTTON_PRESS "$*"; }
START_SPINNER(){ _m SPINNER "start $*"; echo 1; }
STOP_SPINNER(){ _m SPINNER "stop $*"; }
PROMPT_QR(){ _m QR "$*"; }
PROMPT_IMAGE(){ _m IMAGE "$*"; [ -f "$1" ] || _m WARNING "image not found: $1"; }
UI_TAKEOVER(){ _m UI_TAKEOVER; }
UI_RELEASE(){ _m UI_RELEASE; }
DISABLE_DISPLAY(){ _m DISPLAY off; }
ENABLE_DISPLAY(){ _m DISPLAY on; }
RINGTONE(){ _m RINGTONE "$*"; }
ALERT_RINGTONE(){ _m ALERT_RINGTONE; }
VIBRATE(){ _m VIBRATE "$*"; }
DPADLED(){ _m DPADLED "$*"; }
DPADLED_CONFIG(){ _m DPADLED_CONFIG "$*"; }
LED(){ _m LED "$*"; }
BATTERY_PERCENT(){ echo "${MOCK_BATTERY:-80}"; }
BATTERY_CHARGING(){ echo "${MOCK_CHARGING:-0}"; }
GPS_GET(){ echo "${MOCK_GPS:-37.7749 -122.4194 16 0}"; }
GPS_LIST(){ echo "/dev/serial/by-path/mock-gps"; }
GPS_CONFIGURE(){ _m GPS_CONFIGURE "$*"; }
PAYLOAD_GET_CONFIG(){ local f="$MOCK_DIR/config_$1_$2"; [ -f "$f" ] && cat "$f"; }
PAYLOAD_SET_CONFIG(){ _m SET_CONFIG "$*"; printf '%s' "$3" >"$MOCK_DIR/config_$1_$2"; }
PAYLOAD_DEL_CONFIG(){ rm -f "$MOCK_DIR/config_$1_$2"; }
WIGLE_START(){ _m WIGLE_START; echo "$MOCK_DIR/wigle.csv"; }
WIGLE_STOP(){ _m WIGLE_STOP; }
# anything else uppercase (PINEAPPLE_*, WIFI_*, USB_*, ...) just gets logged:
for _c in PINEAPPLE_RECON_NEW PINEAPPLE_HOPPING_START PINEAPPLE_HOPPING_STOP PINEAPPLE_SET_BANDS \
  PINEAPPLE_EXAMINE_BSSID PINEAPPLE_EXAMINE_CHANNEL PINEAPPLE_EXAMINE_RESET PINEAPPLE_LOOT_ARCHIVE \
  PINEAPPLE_SSID_POOL_ADD PINEAPPLE_SSID_POOL_START PINEAPPLE_SSID_POOL_STOP PINEAPPLE_SSID_POOL_CLEAR \
  WIFI_CONNECT WIFI_DISCONNECT WIFI_CLEAR WIFI_WAIT WIFI_PCAP_START WIFI_PCAP_STOP USB_STORAGE USB_WAIT \
  USB_EJECT USB_FREE RESTART_UI DO_A_BARREL_ROLL FIND_CLIENT_IP; do
  eval "$_c(){ _m $_c \"\$*\"; }"
done

# local API: curl --unix-socket /tmp/api.sock http://localhost/api/<route>
curl(){
  local a sock="" url=""
  for a in "$@"; do case $a in */api.sock) sock=1;; http://localhost/api/*) url=${a#http://localhost/api/};; esac; done
  [ -z "$sock" ] && { command curl "$@"; return; }
  _m API "$url"
  local f="${MOCK_API_DIR:-/nonexistent}/${url//\//_}.json"
  [ -f "$f" ] && { cat "$f"; return; }
  case $url in
    pineap/gps/get) set -- ${MOCK_GPS:-37.7749 -122.4194 16 0}
      [[ $MOCK_FW == 1.* ]] && { echo '{}'; return; }
      printf '{"lat":%s,"lon":%s,"alt":%s,"speed":%s,"fix":%s,"connected":true,"sats_tracked":9,"hprecision":1.2}\n' "$1" "$2" "$3" "$4" "${MOCK_FIX:-3}";;
    pager/preferences/get) echo '{"volume":5,"vibrate":true,"lcd_brightness":8,"lcd_timeout":300,"dim_timeout":15,"theme_name":"default"}';;
    pager/preferences/set) echo '{"success":true}';;
    recon/wifi/status) echo '{"pps":120,"total_aps":42,"total_clients":17}';;
    recon/wifi/count) echo '{"aps":42,"clients":17}';;
    *) echo '{}';;
  esac
}
blue5ctl(){ [[ $MOCK_FW == 1.* ]] && return 127
  case $1 in ping) echo pong;; status) echo "scanning";;
    fetch) printf '%-17s %-4s %4s %4s %4s %4s %3s %5s %5s %5s %-10s %s\n' ADDRESS TYPE PWR AVG MIN MAX TX PKTS FIRST LAST INFO NAME \
           AA:BB:CC:00:11:22 LE -61 -64 -70 -58 0 37 12 1 "co 0x004C" "Mock Phone svc=FE9F" \
           DE:AD:BE:EF:00:01 LE -80 -79 -85 -75 0 9 30 4 "co 0x0006" "mfr=0600";;
  esac; }
command -v setsid >/dev/null 2>&1 || setsid(){ "$@"; }      # macOS has no setsid
command -v uci >/dev/null 2>&1 || uci(){ :; }

case "${MOCK_EVENT:-}" in
  handshake) export _ALERT=handshake_captured _ALERT_HANDSHAKE_SUMMARY="Handshake 00:11:22:33:44:55 / 66:77:88:99:AA:BB" \
     _ALERT_HANDSHAKE_AP_MAC_ADDRESS=00:11:22:33:44:55 _ALERT_HANDSHAKE_CLIENT_MAC_ADDRESS=66:77:88:99:AA:BB \
     _ALERT_HANDSHAKE_TYPE=eapol _ALERT_HANDSHAKE_COMPLETE=1 _ALERT_HANDSHAKE_CRACKABLE=1 \
     _ALERT_HANDSHAKE_PCAP_PATH=/root/loot/handshakes/mock.pcap _ALERT_HANDSHAKE_HASHCAT_PATH=/root/loot/handshakes/mock.22000;;
  deauth) export _ALERT=deauth_flood_detected _ALERT_DENIAL_MESSAGE="Deauth flood detected" \
     _ALERT_DENIAL_SOURCE_MAC_ADDRESS=00:11:22:33:44:55 _ALERT_DENIAL_DESTINATION_MAC_ADDRESS=FF:FF:FF:FF:FF:FF \
     _ALERT_DENIAL_AP_MAC_ADDRESS=00:11:22:33:44:55 _ALERT_DENIAL_CLIENT_MAC_ADDRESS=66:77:88:99:AA:BB;;
  auth) export _ALERT=pineapple_auth_captured _ALERT_AUTH_SUMMARY="MSCHAPv2 user1" _ALERT_AUTH_TYPE=mschapv2 \
     _ALERT_AUTH_USERNAME=user1 _ALERT_AUTH_CHALLENGE_IDENTITY=user1;;
  client) export _ALERT=pineapple_client_connected _ALERT_CLIENT_CONNECTED_SUMMARY="66:77:88:99:AA:BB joined TestNet" \
     _ALERT_CLIENT_CONNECTED_AP_MAC_ADDRESS=00:11:22:33:44:55 _ALERT_CLIENT_CONNECTED_CLIENT_MAC_ADDRESS=66:77:88:99:AA:BB \
     _ALERT_CLIENT_CONNECTED_SSID=TestNet _ALERT_CLIENT_CONNECTED_SSID_LENGTH=7;;
  recon_ap|recon_client) export _RECON_SELECTED_AP_BSSID=00:11:22:33:44:55 _RECON_SELECTED_AP_SSID=TestNet \
     _RECON_SELECTED_AP_CHANNEL=6 _RECON_SELECTED_AP_FREQ=2437 _RECON_SELECTED_AP_RSSI=-55 \
     _RECON_SELECTED_AP_ENCRYPTION_TYPE=WPA2 _RECON_SELECTED_AP_CLIENT_COUNT=3 _RECON_SELECTED_AP_HIDDEN=0 \
     _RECON_SELECTED_AP_OUI=MockVendor _RECON_SELECTED_AP_PACKETS=420 _RECON_SELECTED_AP_TIMESTAMP=$(date +%s)
     [ "$MOCK_EVENT" = recon_client ] && export _RECON_SELECTED_CLIENT_MAC_ADDRESS=66:77:88:99:AA:BB \
     _RECON_SELECTED_CLIENT_SSID=TestNet _RECON_SELECTED_CLIENT_BSSID=00:11:22:33:44:55 _RECON_SELECTED_CLIENT_RSSI=-60 \
     _RECON_SELECTED_CLIENT_CHANNEL=6 _RECON_SELECTED_CLIENT_FREQ=2437 _RECON_SELECTED_CLIENT_PACKETS=88 \
     _RECON_SELECTED_CLIENT_OUI=MockPhone _RECON_SELECTED_CLIENT_ENCRYPTION_TYPE=WPA2 _RECON_SELECTED_CLIENT_TIMESTAMP=$(date +%s);;
esac
export DUCKYSCRIPT_USER_DENIED=0 DUCKYSCRIPT_USER_CONFIRMED=1 DUCKYSCRIPT_CANCELLED=2 \
       DUCKYSCRIPT_REJECTED=3 DUCKYSCRIPT_ERROR=4

if [ "${BASH_SOURCE[0]}" = "$0" ] && [ -n "${1:-}" ]; then   # run mode
  : >"$_T"; echo 0 >"$MOCK_DIR/count"; printf '%s\n' "${MOCK_ANSWERS:-}" | tr '|' '\n' | sed '/^$/d' >"$_Q"
  export _PAYLOAD_HOME; _PAYLOAD_HOME="$(cd "$(dirname "$1")" && pwd)"
  _p="$1"; shift
  trap '_m EXIT "status $?"' EXIT
  _m RUN "$_p (firmware $MOCK_FW${MOCK_EVENT:+, event $MOCK_EVENT})"
  source "$_p" "$@"
fi
```
Limits: it can't run MIPS binaries (point the payload at a host-preview build to exercise the
wrapper), it doesn't fake `/sys` files (keep paths like the battery file in variables at the top
so tests can point them at a temp file), and your computer's bash may be older (macOS ships 3.2;
the Pager has 5.2).

### 13.2 Simulate native apps
- C: `cc -DHOST_PREVIEW -o app_host app.c && ./app_host GREEN DOWN DOWN GREEN RED`
- Go: `go run . -preview sim -keys GREEN,DOWN,DOWN,GREEN`

Open the frames and check every screen and state: layout at 480×222, readable text, nothing off
the edge, and Red exits. Then cross-compile and confirm the FPU check prints 0 (C) or that you
used `GOMIPS=softfloat` (Go).

### 13.3 Check and package
`bash pager-check.sh My_Payload general` must end with **0 fail**. It checks:
- header, file names, CRLF line endings, bash syntax, shellcheck;
- takeover without a trap, `$0`-based paths;
- tools the Pager doesn't have (`pkill`, `python`…);
- 2.x incompatibilities (gpspipe/BlueZ only, retuning radios);
- binaries: right architecture, static, **no FPU instructions**;
- missing `chmod +x`, size, README, and obvious secrets.

Then it builds the `dist/` package and install notes.
```bash
#!/usr/bin/env bash
# pager-check.sh - check a Pager payload folder for known problems, then package it for install.
#   bash pager-check.sh path/to/My_Payload [category]     # category default: general
# Exit status: 0 = ready (maybe with warnings), 1 = must fix.
# Produces dist/My_Payload.tar.gz, dist/My_Payload.zip (if zip exists) and dist/INSTALL-My_Payload.txt
set -u
D="${1:?usage: pager-check.sh <payload-folder> [category]}"; CAT="${2:-general}"
D="${D%/}"; NAME=$(basename "$D"); P="$D/payload.sh"
FAIL=0; WARN=0
bad(){ printf '  \033[31mFAIL\033[0m %s\n' "$*"; FAIL=$((FAIL+1)); }
warn(){ printf '  \033[33mWARN\033[0m %s\n' "$*"; WARN=$((WARN+1)); }
good(){ printf '  \033[32m ok \033[0m %s\n' "$*"; }
code(){ grep -v '^[[:space:]]*#' "$P"; }          # payload.sh without comment lines

echo "== $NAME"
[ -f "$P" ] || { bad "no payload.sh in $D"; exit 1; }
[[ "$NAME" =~ ^[A-Za-z0-9._-]+$ ]] && good "folder name" || bad "folder name '$NAME': use only letters, digits, _ - ."
while IFS= read -r f; do bad "space in file name: ${f#$D/}"; done < <(find "$D" -name '* *')

# header
head -1 "$P" | grep -qE '^#!/bin/bash' && good "shebang #!/bin/bash" || bad "first line must be #!/bin/bash"
for k in Title Author Description Version; do
  grep -qE "^# *$k:" "$P" || bad "header missing '# $k:'"; done
grep -qE '^# *Category:' "$P" || warn "header has no '# Category:' (recommended)"

# text hygiene
while IFS= read -r f; do
  if grep -Iq . "$f" 2>/dev/null && grep -q $'\r' "$f"; then bad "Windows line endings (CRLF) in ${f#$D/} - bash will fail on the Pager"; fi
done < <(find "$D" -type f \( -name '*.sh' -o -name '*.txt' -o -name '*.conf' -o -name '*.awk' -o -name '*.view' -o -name 'payload*' \))
bash -n "$P" 2>/tmp/pc.$$ && good "bash syntax" || { bad "bash syntax error: $(head -3 /tmp/pc.$$)"; }
rm -f /tmp/pc.$$
if command -v shellcheck >/dev/null; then
  n=$(shellcheck -s bash -S warning -e SC2034,SC2154,SC1091 "$P" 2>/dev/null | grep -c '^In ')
  [ "$n" = 0 ] && good "shellcheck (warnings)" || warn "shellcheck: $n warning(s) - run: shellcheck -s bash $P"
fi

# runtime rules
if code | grep -q 'UI_TAKEOVER'; then
  code | grep -qE 'trap .*UI_RELEASE|trap +[A-Za-z_]+ ' && good "UI_TAKEOVER has a trap" \
    || bad "UI_TAKEOVER without a trap that runs UI_RELEASE (screen stays frozen on errors)"
fi
if code | grep -qE '\$\(dirname "?\$0|\$\{?BASH_SOURCE|cd "?\$\(dirname' && ! grep -q '_PAYLOAD_HOME' "$P"; then
  warn "finds files via \$0/dirname - menu launches run a /tmp copy; use \$_PAYLOAD_HOME first"
fi
for t in pkill nohup python python3 perl lua bc xxd socat inotifywait ffmpeg; do
  code | grep -qE "(^|[^A-Za-z0-9_./-])$t( |$)" && bad "uses '$t' - not installed on the Pager"
done
if code | grep -qE 'bluetoothctl|hcitool|btmgmt|hciconfig'; then
  grep -q blue5ctl "$P" && good "BlueZ use has a blue5 path" \
    || warn "uses BlueZ tools - on firmware 2.x blue5 owns Bluetooth and bluetoothctl hangs; add a blue5ctl path"
fi
if code | grep -q 'gpspipe'; then
  grep -q 'pineap/gps/get' "$P" && good "gpsd use has a 2.x GPS API path" \
    || warn "uses gpspipe - gpsd is off on firmware 2.x; read /api/pineap/gps/get too"
fi
code | grep -qE 'iw (phy|dev) [^ ]+ (interface add|set channel)' && \
  warn "creates/retunes Wi-Fi interfaces - on 2.x sniff pineapd's existing monitor interfaces instead"
code | grep -q 'leds/buzzer' && ! code | grep -qE 'trap' && warn "writes the buzzer but has no trap to silence it on exit"
code | grep -qE 'pgrep -f|killall' && warn "pgrep -f/killall can match your own shell; prefer PID files / \$!"
grep -nE '(password|passwd|api_?key|token|secret)[[:space:]]*=[[:space:]]*["'"'"']?[A-Za-z0-9]{6,}' "$P" | grep -viE 'placeholder|changeme|your_|example' >/dev/null \
  && warn "looks like a real secret in payload.sh - ship placeholders"

# binaries
if command -v python3 >/dev/null; then
  while IFS= read -r f; do
    r=$(python3 -I - "$f" <<'PY'
import struct,sys
b=open(sys.argv[1],'rb').read()
if b[:4]!=b'\x7fELF': print('notelf'); sys.exit()
cls,data=b[4],b[5]; mach=struct.unpack('<H',b[18:20])[0]
if cls!=1 or data!=1 or mach!=8: print('wrongarch'); sys.exit()
phoff,=struct.unpack('<I',b[28:32]); phes,phn=struct.unpack('<HH',b[42:46])
interp=False; fpu=0
for i in range(phn):
    o=phoff+i*phes; t,off,va,pa,fs,ms,fl=struct.unpack('<IIIIIII',b[o:o+28])
    if t==3: interp=True
    if t==1 and fl&1:
        for j in range(off,off+fs-3,4):
            w,=struct.unpack('<I',b[j:j+4]); op=w>>26
            if op in (0x31,0x35,0x39,0x3d) or (op==0x11 and ((w>>21)&31) in (0,4,16,17,20,21)): fpu+=1
print(('dynamic' if interp else 'static'),fpu)
PY
)
    case $r in
      notelf|"") ;;
      wrongarch) bad "${f#$D/}: not a 32-bit little-endian MIPS binary (build for mipsel / GOARCH=mipsle)";;
      dynamic*) warn "${f#$D/}: dynamically linked - only works if every library exists on the Pager (musl soft-float); prefer static";;
      static*) n=${r#static }
        if [ "$n" -gt 0 ]; then bad "${f#$D/}: contains ~$n FPU instructions - will die with 'Illegal instruction' (use integer math / GOMIPS=softfloat)"
        else good "${f#$D/}: static mipsel, no FPU instructions"; fi
        [ -x "$f" ] || warn "${f#$D/} isn't executable; also add 'chmod +x \"\$DIR/$(basename "$f")\"' in payload.sh (web uploads drop the bit)"
        grep -q "chmod +x" "$P" || warn "payload.sh doesn't chmod +x its binary - installs via browser upload lose the executable bit";;
    esac
  done < <(find "$D" -type f ! -name '*.sh' ! -name '*.md' ! -name '*.txt' -size +100c)
else
  warn "python3 not found - skipped binary checks"
fi

# size
KB=$(du -sk "$D" | awk '{print $1}')
[ "$KB" -gt 8192 ] && warn "folder is $((KB/1024)) MB - big for the Payload Portal (~60 s download limit); drop large media"
[ -f "$D/README.md" ] && good "README.md present" || warn "no README.md (needed for the Hak5 repo)"

echo "== result: $FAIL fail, $WARN warn"
[ "$FAIL" -gt 0 ] && exit 1

# package
OUT="$(cd "$(dirname "$D")" && pwd)/dist"; mkdir -p "$OUT"
chmod +x "$P"
( cd "$(dirname "$D")" && COPYFILE_DISABLE=1 tar -czf "$OUT/$NAME.tar.gz" --exclude='.DS_Store' "$NAME" )
command -v zip >/dev/null && ( cd "$(dirname "$D")" && rm -f "$OUT/$NAME.zip" && zip -qr "$OUT/$NAME.zip" "$NAME" -x '*.DS_Store' )
DEST="/root/payloads/user/$CAT/$NAME"
cat >"$OUT/INSTALL-$NAME.txt" <<EOF
Install $NAME on a WiFi Pineapple Pager (category: $CAT)

A) From a computer with a terminal (Pager connected by USB-C, address 172.16.52.1):
   scp $NAME.tar.gz root@172.16.52.1:/tmp/
   ssh root@172.16.52.1 'mkdir -p /root/payloads/user/$CAT && tar -xzf /tmp/$NAME.tar.gz -C /root/payloads/user/$CAT && chown -R root:root $DEST && chmod +x $DEST/* && rm /tmp/$NAME.tar.gz'

B) Browser only (firmware 2.x): open http://172.16.52.1:1471 , log in, go to Files,
   open /root/payloads/user/$CAT/ , create a folder named $NAME , open it and upload every file
   from the $NAME folder (not the .zip). payload.sh fixes executable permissions itself.

Then on the Pager: Payloads -> user -> $CAT -> $NAME.
To remove it: delete $DEST
EOF
echo "== packaged: $OUT/$NAME.tar.gz $( [ -f "$OUT/$NAME.zip" ] && echo "$OUT/$NAME.zip") $OUT/INSTALL-$NAME.txt"
command -v shasum >/dev/null && (cd "$OUT" && shasum -a 256 "$NAME.tar.gz") || (cd "$OUT" && sha256sum "$NAME.tar.gz" 2>/dev/null)
exit 0
```

### 13.4 Other checks
- `bash -n payload.sh`, plus `shellcheck -s bash payload.sh` if installed.
- Test JSON parsing locally with `jq` against saved samples of the API responses in section 8.8.
- Text in your own UI: count characters against your font (5×7 at 1× ≈ 80 columns; 7×13 ≈ 68).
  The Pager's own small UI font is about 9 px per character on hardware.

### 13.5 What offline testing cannot prove
Real button feel and timing, radio/GPS/Bluetooth behaviour, how the firmware queues dialogs,
performance on the slow CPU, and buzzer volume. Say so in the handover.

### 13.6 The on-device test checklist (give this to the user)
Tailor it, but always include:
1. Install (section 12) and confirm the payload shows in the menu.
2. Launch it. Note what the screen shows first (a photo is fine).
3. Walk every menu option and button, including Red/back on every screen.
4. Exit normally and confirm the Pager returns to its menus (screen not frozen, sound off).
5. For watchers: run once (ON), trigger the condition if possible, run again (OFF), and check
   that it really stopped.
6. Run it a second time to check that a fresh start works.
7. If anything fails, send the payload log text, a photo, and (on 2.x via the Virtual Pager
   terminal) the output of `logread | tail -50`.

---

## 14. Testing on a device safely (user or agent)

- **If the screen freezes or the UI stops responding:** restart the UI with
  `/etc/init.d/pineapplepager restart` (terminal), or reboot the Pager (hold Power). Recon
  recovers by itself. After a UI restart a "welcome to version…" popup appears and swallows the
  first button press.
- **Don't kill a payload in the middle of a takeover.** That can leave the UI's takeover API
  hanging. Exit through the app's own Red → exit path.
- **Raise the screen timeout while testing** (Settings, or `pager/preferences/set` with
  `lcd_timeout`/`dim_timeout`). After the display dims, scripted input may stop updating the UI.
- **USB peripherals and DTR:** opening and closing USB-serial devices repeatedly resets them and
  forces USB re-enumeration. Doing that in a tight loop has crashed a host computer's USB stack.
  Batch serial work into one open session, unplug peripherals before big copies, and **stop when
  the connection starts glitching.**
- **SDR dongles:** `rtl_power` only writes on a clean exit, and killing it mid-sweep can wedge the
  tuner until it's replugged. Use one-shot sweeps (`-1`). Never SIGKILL it.
- **Back up anything you change** outside your payload folder, and say how to undo it.

---

## 15. Serial / UART peripherals

Plug the device into the **USB-A host port**. Most adapters enumerate as `/dev/ttyUSB*` or
`/dev/ttyACM*`. Stable names are in `/dev/serial/by-path/`. Find a device by its product name in
sysfs (`/sys/bus/usb/devices/*/product`) instead of guessing the tty number.

```bash
DEV=/dev/ttyUSB0
stty -F $DEV 115200 cs8 -cstopb -parenb -echo -crtscts clocal -hupcl raw
exec 3<>$DEV            # open ONCE and keep it open for the whole session
sleep 2; timeout 1 cat <&3 >/dev/null   # flush boot noise
printf 'HELLO\r\n' >&3
timeout 2 cat <&3
exec 3>&-
```
Hard-won rules:
- **Opening the port toggles DTR, and many boards (CP2102/CH340 designs) reset on DTR.** Open once
  and hold the descriptor. Use `-hupcl` so close doesn't drop the lines, and `clocal` so reads
  don't wait for carrier.
- **The GPS stack may grab your port.** gpsd (1.1.x), or pineapd's GPS autodetect (2.0), can claim
  a USB-serial device, probe it at GPS baud rates and toggle DTR. On 1.1.x stop gpsd
  unconditionally before using the port (`/etc/init.d/gpsd stop`; checking first with `pgrep`
  races procd's respawn) and restart it after. On 2.0, check `uci get pineapd.@pineapd[0].gpsdevice`
  isn't your device. Probing at 1200 baud can even reboot some boards into their bootloader.
- In native code, keep `VTIME` small (≤0.2 s) and do your waiting in a deadline loop. A large
  `VTIME` with `VMIN=0` makes every idle read block for the full timeout.
- Expect garbage on the first read after opening the port. Sync on a known prompt before parsing.
- There's no terminal program on the device. Use `stty` + redirection, or write a tiny tool.

---

## 16. Common mistakes (checklist)

- [ ] Used `$0`/`dirname $0` to find bundled files → **use `$_PAYLOAD_HOME`** (the payload runs as a `/tmp` copy).
- [ ] Called `UI_TAKEOVER` only inside the binary → call it in `payload.sh`, with a `trap` for `UI_RELEASE`.
- [ ] The launch keypress triggered an action → drain input for the first ~200 ms.
- [ ] Floats in C, or a glibc/hard-float binary → `Illegal instruction`. Freestanding C, or Go with `GOMIPS=softfloat`.
- [ ] Forgot the screen is rotated → `index = x*222 + (221-y)`.
- [ ] Drew at 60 fps → the panel does 20; pace it or draw on change.
- [ ] Mixed up Green/Red → Green = 305 (OK), Red = 304 (back).
- [ ] `gpspipe` or `bluetoothctl` on 2.0 → pineapd GPS API and `blue5ctl`. (bluetoothctl hangs.)
- [ ] Retuned or created monitor interfaces on 2.0 → sniff pineapd's existing ones passively.
- [ ] Watcher can't be stopped quickly → `setsid` + `kill -TERM -PID`.
- [ ] `ALERT` inside a `LIST_PICKER` loop never appears → use `PROMPT`.
- [ ] Expected a new alert event type → only the four Wi-Fi events exist; use a watcher.
- [ ] Wanted a theme icon to launch the payload → not possible on current firmware.
- [ ] Buzzer left on after exit → write `0` to `/sys/class/leds/buzzer/brightness` in the trap.
- [ ] `pkill`/`nohup`/`python`/`bc` in the payload → not installed. Use `pgrep`+`kill`, `setsid`, `awk`.
- [ ] `$(…)` in a hot loop → forks are slow here; use fork-free bash.
- [ ] `pgrep -f` matched itself and killed your shell → PID files, or the `[x]` trick.
- [ ] `local a=$1 b=$((a+1))` → `b` sees an empty `a`. Split it into two `local` lines.
- [ ] Hard-coded personal values (SSIDs, keys, URLs, your home GPS) → use placeholders.
- [ ] Binary not executable after a browser upload → `chmod +x` it in `payload.sh`.
- [ ] CRLF line endings from a Windows editor → `pager-check.sh` catches them; save as LF.
- [ ] Recon variables compared without trimming whitespace → trim first.
- [ ] Called `WIGLE_START` while logging was already on → a second log file; check first, and only stop what you started.
- [ ] Network call in the keypress path → fetch in the background; draw from a cache.
- [ ] Assumed the agent could test on the device → test offline, hand over a checklist (13.6).
- [ ] Assumed it works → say what was verified on hardware and what wasn't.

---

## 17. Sharing a payload (hak5/wifipineapplepager-payloads)

- Put it at `library/user/<existing-category>/<Payload_Name>/`, with no spaces, `payload.sh` as the
  entry point, and the header fields (Title, Author, Description, Version, Category).
- **Include everything it needs to run.** For native code, ship the **prebuilt binary** alongside
  its **source and `build.sh`**, and record the compiler version, flags and the binary's
  **SHA-256** in the README so reviewers can rebuild it byte-for-byte.
- `README.md`: what it does, controls, settings, files, the firmware version you tested on, and
  build/verify steps.
- Use placeholders for anything user-specific. Cite code you build on. Test on a real Pager.
- Fork the repo, make one clean commit per payload, and open a PR against `master`. Once it's in
  review, push follow-up fixes as new commits.

---

## 18. Optional: when the agent has SSH to the Pager

Only use this if the user has set up SSH key login for you (`ssh root@172.16.52.1` works without
a prompt). Everything above works without it. Ask before doing anything disruptive, and leave
the device as you found it.

- **Probe once, up front:** `PAGER=172.16.52.1 bash pager-doctor.sh` (firmware, Bluetooth/GPS
  mode, free space, categories).
- **Install:** use the Option A commands in section 12. Verify with `sha256sum` against your
  build.
- **Run a native binary briefly without a takeover** to check drawing, and grab the screen
  mid-run (no risk of wedging the UI):
  ```bash
  ssh root@172.16.52.1 '(timeout 3 /tmp/app >/dev/null 2>&1 &) ; sleep 1.5; cat /dev/fb0' > fb.raw
  python3 fb2png.py fb.raw shot.png        # then look at shot.png
  ```
  The Pager UI may repaint over parts of it; that's expected.
- **Run a payload from the shell** with the takeover stubbed, so killing it can't wedge the UI:
  `UI_TAKEOVER(){ :; }; UI_RELEASE(){ :; }; export -f UI_TAKEOVER UI_RELEASE` before
  `bash payload.sh`. Do the final test from the Payloads menu (ask the user to press the buttons).
- **Logs:** `logread | tail -100`, `logread | grep -i <YourTag>`; for live output use
  `timeout 30 logread -f`.
- **Injecting button presses** (for unattended UI tests) needs a tiny native helper that writes
  16-byte `input_event` records (press, `EV_SYN`, release, `EV_SYN`) to `/dev/input/event0`. The
  UI may stop reacting to injected input after it dims; raise the timeouts first.
- **Your own shell can be a target.** Never `pgrep -f`/`kill` with a pattern that appears in your
  own SSH command line. Put cleanup in a script file on the device and run `sh /tmp/cleanup.sh`.
- **The USB link drops sometimes** (ssh exit 255). Keep commands short, wrap them in `timeout`,
  retry, and after a drop check for leftovers (`ps | grep [m]yapp`). `setsid` processes survive
  drops.
- **Restore what you changed:** screen timeouts, stopped services, test folders under
  `/root/payloads`, and files in `/tmp`.
