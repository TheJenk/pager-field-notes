# Capabilities: what's installed & supported

A map of what ships on the device (firmware 1.1.3) so you know what you can call from a payload and
what peripherals/filesystems "just work." ~354 opkg packages are installed.

## Scripting & languages (on device)
- **bash** (full, not just ash), `coreutils`, `awk`, `jq`, `sqlite3`, `openssl`, `git`, `nano`, `file`.
- **No compilers or interpreters** — no gcc/cc, no python/perl/lua/go. Compile native/Go tools on a
  host and copy them over. On-device logic = bash + awk + jq (+ sqlite3 for structured data).

## Networking & Wi-Fi tools
- `iw`, `iwinfo`, `wpa_supplicant`/`wpad`, `hostapd`/`hostapd_cli`, `dnsmasq`, `nft`/`fw4`, `tcpdump`,
  `nmap`, `netcat`, `macchanger`, `ethtool`, `curl`/`wget`/`uclient-fetch`, `openvpn`, `wireguard`/`wg`,
  `autossh`, `ssh`/`sshd`/`scp`, `whoismac`.
- **Handshake/PMKID toolkit** (hcxtools): `hcxpcapngtool`, `hcxhashtool`, `hcxpmktool`, `hcxpsktool`,
  `hcxwltool`, `hcxhash2cap`, `wlancap2wpasec`.
- Packet capture via `tcpdump` (and `libpcap` is present for your own tools).

## Bluetooth
**2.0.0+:** the `blue5` scanner owns the adapter and stops `bluetoothd`. Read devices with
`blue5ctl fetch` or the `recon/bluetooth/*` API; see [What changed in 2.0.0](../firmware/2.0.0.md).
The BlueZ notes below are for 1.1.x.

BlueZ (1.1.x) full stack: `bluetoothd`, `bluetoothctl`, `btmgmt`, `hcitool`, `hciconfig`, `gatttool`, `sdptool`,
`rfcomm`, `l2ping`, `hcidump`, `btmon`, over D-Bus. BLE/BT scanning and GATT are possible. Note: in a
payload context prefer `bluetoothctl` — `btmgmt`/`hciconfig` tend to hang fighting `bluetoothd`.

## Storage & filesystems
- USB mass storage works (`usb_storage`, `sd_mod`, `scsi_mod`); tools `blkid`, `lsblk`, `fdisk`/`gdisk`/
  `parted`, `mkfs.*`, `mount`/`ntfs-3g`.
- Filesystem drivers loaded: **ext4, vfat/fat, exfat, ntfs/ntfs3, cifs (SMB)**. So you can read/write
  most USB drives and even mount SMB shares.

## USB device support (drivers loaded)
- **USB-serial:** `cdc_acm`, `cp210x`, `ftdi_sio`, `pl2303`, `ch341`/`ch348`, plus `usbserial` with
  qcserial, sierra, keyspan, oti6858, mct_u232, garmin, option/usb_wwan (cellular modems). Almost any
  UART bridge → `/dev/ttyACM*`/`ttyUSB*`.
- **USB-Ethernet:** `asix`/`ax88179`/`ax88796b`, `r8152`, `cdc_ether`/`cdc_ncm`/`cdc_eem`,
  `rndis_host`, `lan78xx`, `aqc111`, and many legacy (`dm9601`, `smsc95xx`, …). So USB NICs and
  phone-tethering/RNDIS generally work.
- **USB Wi-Fi:** the mt76 family (`mt7601u`, `mt76x0u`, `mt76x2u`, **`mt7921u`**) — external MediaTek
  Wi-Fi dongles are supported, including modern MT7921 (Wi-Fi 6/6E).
- **HID:** `usbhid`/`hid_generic`; input via `evdev`/`uinput`; `gpio_keys` for the hardware buttons.

## Framebuffer / fonts / media
- `fbi` and `fbtest` can display images directly to `/dev/fb0` — a payload can show a JPEG/PNG without
  writing its own renderer. `exiftran` for JPEG transforms.
- Libraries present for your own tools: `libfreetype`, `libfontconfig` (+ `fc-*` tools), `libjpeg`,
  `libpcap`, `libcurl`, `libcrypto`/`libmbedcrypto`, `libnl*`, `libnftnl`, `libsqlite`.
- **Fonts:** system `DejaVuSans.ttf`; the UI's own fonts are **`Steelfish`** (the Pager UI typeface)
  and **`MaterialIcons-Regular`** (glyphs). Useful if a takeover payload wants to match the UI look.

## Other
- `ubus`/`ubusd` (OpenWrt IPC bus), `uci` (config), `logread`/`logd`, `procd`/`service`, `opkg`
  (package manager), `sysupgrade`, `mtd`, `i2c*` tools (there's an I2C bus, `10000900.i2c`).
- GPS stack: `gpsd`, `gpspipe`, `gpsctl`, `gpsdecode`, `cgps`/`gpsmon`, `gpxlogger`, `lcdgps`.
