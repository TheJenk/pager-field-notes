# Display & Input

## Framebuffer
- `/dev/fb0`, **222 × 480, 16 bits/pixel RGB565**, stride 444 bytes (no padding → 222×2).
- The panel is **landscape 480 × 222** and rotated relative to the framebuffer memory. For a landscape
  pixel `(lx, ly)` with `lx` in `0..479` and `ly` in `0..221`, the framebuffer index is:

  ```
  fb_index = lx * 222 + (221 - ly)
  ```

  So memory is organized as 480 columns of 222 pixels, with each column's rows flipped. Use this both
  to draw and to decode a captured frame.
- Panel refresh is about **20 fps** — draw at that rate or slower; faster is wasted.
- The panel controller is an **ST7796U** on SPI (fbtft driver). The rotation and the 20 fps cap
  come from its device-tree settings (`rotation` 90, `fps` 20). The firmware's own boot splash files
  (`/lib/pager/*.fb`) are raw frames in this same rotated layout.
- Colour is little-endian RGB565: `(r & 0xF8) << 8 | (g & 0xFC) << 3 | b >> 3`. No red/blue swap.
- **Backlight:** `/sys/class/backlight/backlight_pwm/brightness` (0–16). The screen still dims and
  sleeps on the user's timeout during a takeover.
- **Capturing a frame** (for debugging a UI): read the raw device and convert it. Over SSH:
  `cat /dev/fb0 > fb.raw`, then
  `ffmpeg -f rawvideo -pixel_format rgb565le -video_size 222x480 -i fb.raw -vf transpose=2 out.png`.
  Without ffmpeg, apply the rotation formula yourself. On a slow link, `gzip -c /dev/fb0 | base64`
  and decode on the host. A USB link manages about 5–6 raw frames per second for video.
- **Drawing without a compiler:** `fbi` can show a JPEG/PNG on `/dev/fb0`, and `PROMPT_IMAGE` shows
  one as a modal. Pure-bash drawing works by writing 444-byte rows (one upright column each) with
  `dd … conv=notrunc`, but it's only fast enough for coarse, slow-changing displays.

## Taking over the screen — *verified*
`UI_TAKEOVER` makes the Pager app stop drawing and stop handling buttons; `UI_RELEASE` hands them
back. Rules learned on hardware:
1. Call both **from `payload.sh`**, not only from inside your binary. Otherwise the payload log
   screen keeps redrawing and the desktop flashes through.
2. `trap` `UI_RELEASE` on every exit path, and run it **first** in cleanup so a slow cleanup step
   can't leave the screen frozen.
3. Draw a first frame **immediately**, before any slow setup.
4. **Drain the launch keypress.** The button press that started the payload is often still queued,
   so discard input for the first ~150–400 ms.
5. Don't `SIGSTOP` the UI process or poke the console (an older technique). The takeover is the
   sanctioned handoff.

## Input (buttons)
- `/dev/input/event0` (device name `keys`). Standard Linux evdev. On this 32-bit kernel each
  `struct input_event` is **16 bytes**: 8-byte timestamp, `u16 type`, `u16 code`, `s32 value`.
- **Key codes:**

  | Button | Code |
  |---|---|
  | Up | 103 |
  | Down | 108 |
  | Left | 105 |
  | Right | 106 |
  | Power | 116 |
  | "Red" | **304** |
  | "Green" | **305** |

  Note the device reports **Red = 304** and **Green = 305**, which is the *opposite* of the usual
  `BTN_SOUTH` (304) / `BTN_EAST` (305) convention — don't assume. In practice **Green is
  select/confirm** and **Red is back/exit**.
- If you leave the **Power** key unhandled in a takeover payload, the system still does its normal
  display-sleep (evdev broadcasts to all readers).
- `value` is 1 for press, 0 for release and 2 for auto-repeat while held. For a **long press**, time
  press-to-release (~600 ms works well), and fire tap actions on release if a button also has a hold
  action.
- The device-tree labels for the two buttons (`a-button`/`b-button`, `BTN_SOUTH`/`BTN_EAST`) don't
  map cleanly to colours, so go by the codes above.
- **Injecting input** (for automated testing) is possible by writing EV_KEY press + EV_SYN +
  EV_KEY release + EV_SYN records to `event0`. That needs a small native helper. After the display
  dims or a modal opens, injected input may stop updating the UI; raise the screen timeouts first.
