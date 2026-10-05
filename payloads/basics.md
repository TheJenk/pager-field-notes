# Payload basics

A payload is a folder containing `payload.sh` (the entry point, with a header comment) plus any
resources. Payloads run as root. They can be plain shell, or drive an on-screen UI by taking over the
framebuffer.

## Where payloads live
```
/root/payloads/
  alerts/<event>/          # event-driven alert payloads (fixed event names)
  recon/...
  user/<category>/<Name>/  # most payloads: user/general/<Name>, etc.
```

## `payload.sh` header
Begin with comments giving at least Title and Author; add Description, Version, Category:
```sh
#!/bin/bash
# Title: Example
# Author: You
# Description: What it does.
# Version: 1.0
# Category: general
```
Use **placeholders** for anything user-specific (URLs, keys, SSIDs) — never ship personal values.

## Execution models
- **Normal payload** — runs when launched from the Payloads app.
- **Alert payloads** are event-driven: the folders under `/root/payloads/alerts/` correspond to a
  **fixed set of events** the daemon fires (deauth flood detected, handshake captured, auth captured,
  client connected). You **cannot add a new event type** (e.g. "GPS disconnected"). For "alert me when
  X" where X isn't one of those events, use a **background watcher** instead (see
  [patterns](patterns.md)).
- **Boot payload** — one payload can be set to run once at boot.

## Launching programmatically
The UI's own launch path is the local API `payloads/{id}/run_payload` (numeric payload ID). Note that a
**theme UI element cannot launch a payload** on current firmware — see
[firmware findings](../firmware/findings.md).

## On-screen payloads
To draw your own UI, the `payload.sh` wrapper calls `UI_TAKEOVER`, runs a small native/Go program that
writes `/dev/fb0` and reads `/dev/input/event0`, and calls `UI_RELEASE` on exit (via a `trap`). See
[building native & Go tools](building.md).
