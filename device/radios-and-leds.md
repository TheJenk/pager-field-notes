# Radios, LEDs & I2C

## Wi-Fi radios (two)
The Pager has **two** 802.11 radios:

- **phy0** — the SoC's internal MAC (`mediatek,mt7628-wmac`, mt76 driver). 2.4 GHz.
- **phy1** — an **MT7921U** (USB, VID:PID `0E8D:7961`, `mt7921u` driver) — **tri-band: 2.4, 5 and
  6 GHz (Wi-Fi 6/6E)**. (The internal Wi-Fi temperature shows up as `hwmon` `mt7921_phy1`.)

Across the radios the device advertises 2.4, 5 **and 6 GHz** bands (lots of channels). Supported
interface modes include **managed, AP, AP/VLAN, monitor, mesh point, IBSS, P2P-GO/P2P-client** — so
both rogue-AP and monitor/recon workflows are available. External MediaTek USB Wi-Fi dongles can be
added on the host port (mt76 USB drivers are loaded).

Interfaces you'll typically see: a management/AP interface, a client interface (e.g. `wlan0cli` for
upstream Wi-Fi), and monitor interfaces (`wlanXmon`). Use `iw dev` to enumerate and `iw phy` for
capabilities.

## LEDs
Addressable LEDs under `/sys/class/leds/` (also driven by the `LED` / `DPADLED` /
`DPADLED_CONFIG` Hak5 commands):

- **D-pad RGB:** `up-led-{red,green,blue}`, `down-led-*`, `left-led-*`, `right-led-*` — a full RGB LED
  behind each direction.
- **Buttons:** `a-button-led`, `b-button-led`.
- **Buzzer:** `buzzer` (a `leds`-class PWM tone device — see [sensors](sensors.md)).
- `mt76-phy0` (radio activity LED).

A payload can give rich colour feedback on the d-pad/buttons (write `brightness`/colour via the LED
class, or use the `LED`/`DPADLED` commands).

## I2C
There's an I2C bus (`i2c-0`, `10000900.i2c`) with the usual tools (`i2cdetect`, `i2cget`, `i2cset`,
`i2cdump`, `i2ctransfer`) — relevant if you probe/expand onboard peripherals.
