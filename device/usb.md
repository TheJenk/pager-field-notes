# USB

The Pager has two distinct USB roles:

## USB-C — device/gadget mode (the computer link)
The USB-C port runs in **device/gadget** mode: it presents **Ethernet-over-USB** to a connected
computer (so you get an SSH-able network link), a serial console, and charging. Because it's in device
mode, **it cannot also host a peripheral.** This is the port you use to reach the device from a
computer.

## Host port — for peripherals
There is a **USB host port** (fronted by an internal USB 2.0 hub) where peripherals attach — GPS
receivers, Wi-Fi dongles, serial/UART devices, etc. They enumerate on the internal bus (as `1-1.x`).

To connect a **UART / serial peripheral** while the USB-C link is busy with your computer, use this
**host port**. It's **USB-A**, so a **USB-C peripheral needs a C-to-A adapter**. *Verified:* the
USB-A host port and the USB-C computer link **work at the same time**, so a peripheral can be
plugged in while you stay connected. Mind the **power budget**: GPS sticks, Wi-Fi dongles, SDRs,
small microcontroller boards and even a small Linux gadget ran fine, but test anything hungrier, or
use a powered hub.

**Serial behaviour:** opening a USB-serial port toggles DTR, and many boards reset on DTR, so hold
one open descriptor for the whole session. The GPS stack may also probe new serial devices. See
[patterns](../payloads/patterns.md#serial--uart-peripherals--verified).

## USB-serial driver support
A broad set of USB-serial drivers is loaded, so almost any common UART bridge enumerates as
`/dev/ttyACM*` or `/dev/ttyUSB*`:

- `cdc_acm`, `cp210x`, `ftdi_sio`, `pl2303`, `ch341`
- `usbserial` bound to many more (qcserial, sierra, keyspan, oti6858, mct_u232, garmin, …)

So plugging in a serial gadget and getting a `/dev/tty*` generally "just works." There's **no terminal
program** on the device, though — interact via `stty` + read/write redirection, or ship a small
terminal of your own.

> Tip: swapping peripherals on the **host** port doesn't disturb the **USB-C** computer link, so an
> SSH session over USB-C stays up while you hot-swap a device on the host port.
