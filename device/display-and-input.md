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
- **Capturing a frame** (for debugging a UI): read the raw device and reshape it, e.g.
  `gzip -c /dev/fb0 | base64` over a link, then on your host decode base64 → gunzip → interpret as
  222×480 RGB565 and apply the rotation formula to get a 480×222 image.

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
- **Injecting input** (for automated testing) is possible by writing EV_KEY press + EV_SYN +
  EV_KEY release + EV_SYN records to `event0`.
