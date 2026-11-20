# Firmware findings

Observations from testing on firmware **1.1.3** (`24.10.1`). "Verified" means reproduced on hardware.
For what changed in **2.0.0**, see [What changed in 2.0.0](2.0.0.md).

## You cannot launch a payload or run a command from a theme item (1.1.3) — *verified*
A theme menu/dashboard item's `target` is dispatched by the UI as a **component/screen name to navigate
to**. Targets that are meant to run a command or launch a payload fall through and fail:

- `execute_payload_<shortcut>` and `execute-command-<name>` both log
  **`[UI/GENERIC] Theme targets component that does not exist: <target>`** and do nothing — tested from
  the dashboard *and* from a registered menu.
- The `_unused.payload_shortcuts` and `_unused.commands` maps still ship in the stock theme, and the
  handler code (an execute-command component and a payload-shortcut component) is still compiled into
  the UI binary — the **symbols are present and identical in 1.1.2 and 1.1.3**, and the stock theme's
  top-level keys are identical between the two versions. An older community theme uses the
  `execute_payload_<name>` / `execute_<command>` target syntax, which indicates this was a working
  feature that **regressed** (the handler is present but the target no longer routes to it).

**Implication:** on current firmware you can't make, say, a desktop icon launch a payload. Payloads are
launched from the Payloads app (on 2.0.0 the `payloads/{id}/run_payload` route doesn't work on the
Pager either; see [local API](../device/local-api.md)). If you need "tap a thing →
run a payload", it currently requires either Hak5 restoring the routing, or patching the UI binary
(impractical: a large stripped Go MIPS binary; a bad patch bricks the UI, recoverable only over SSH).

Registries that *do* work for navigation: a custom screen must be listed in `theme.json`
(`generic_menus` for dashboards/menus, `option_dialogs` for value pickers) — both accept added keys —
or it's "component does not exist". An invisible-overlay item (1×1 transparent normal layer + a
highlight box as `selected_layers`) can make a decorative spot focusable, and z-order over the firmware
status bar is fine — but since it can only *navigate*, it's limited by the above.

## `GPS_LIST` stays stale after an unplug — *verified* (and the root cause)
Unplug a USB-serial GPS and `GPS_LIST` keeps reporting it; restarting gpsd doesn't clear it. Chain:

1. `GPS_LIST` reads the UI's cached `system/serial/list`, which enumerates `/dev/serial/by-path/`.
2. On unplug the kernel removes the real `ttyACM*`/`ttyUSB*` node, but the `/dev/serial/by-path/<id>`
   **symlink is left behind, dangling**.
3. So the cached list still "sees" a device that's gone.

Root cause: the stock hotplug script `/etc/hotplug.d/usb/10-tty-naming` computes the symlink name
**differently in its `unbind` branch than its `bind` branch** (strips a different path prefix and omits
the `-port` suffix), so its `readlink`-based removal misses and the stale symlink is never deleted.

### Additive fix (own code; overlay-persistent, reversible)
Drop this in as `/etc/hotplug.d/usb/99-serial-cleanup` — it sweeps dangling `/dev/serial/*` symlinks on
unplug, without editing the stock script:

```sh
# /etc/hotplug.d/usb/99-serial-cleanup
[ "$ACTION" = "unbind" ] || [ "$ACTION" = "remove" ] || exit 0
for l in $(find /dev/serial -type l 2>/dev/null); do [ -e "$l" ] || rm -f "$l"; done
```

`hotplug-call` sources each `/etc/hotplug.d/<subsystem>/*` script in its **own subshell**, so a later
`99-*` script still runs even though the stock scripts use `exit`. Verified: after adding this, an
unplug clears the stale `by-path` link and `GPS_LIST` correctly returns "No supported serial GPS devices
found" (and it survives reboot via the overlay). ⚠️ Modify device files at your own risk; keep a copy,
since a firmware update may revert it.

### Detecting GPS loss in a script (the robust way)
Don't poll `GPS_LIST`. Either watch **gpsd data liveness** (no `TPV`/`SKY` for a few seconds = gone) or
test the by-path node with `[ -c "$(uci -q get gpsd.core.device)" ]`. gpsd may not auto-resume after a
replug; `/etc/init.d/gpsd restart` reattaches it. (See [payload patterns](../payloads/patterns.md).)

## Startup welcome alert eats input after a UI restart — *verified*
After the UI service restarts, a "Welcome to version …" modal appears a few seconds later and absorbs
button presses. When automating navigation, wait for it and dismiss it (A/B) before sending keys.

## Launch internals (for reference)
- UI → daemon launch path exists (`ipc_manager.launch_payload`, taking a payload identifier + arg
  list) and the API route `payloads/{id}/run_payload` (payload IDs are numeric).
- The confirm dialog you normally see when launching a payload is the `launch_payload_dialog`
  component; the actual run is `launch_payload`, which relies on the payload context set by the browse
  flow — so you can't point it at an arbitrary payload from an unrelated screen.

## Overlay filesystem (how edits persist)
Root is `squashfs` (`/rom`, read-only, the signed image) + `jffs2` (`/overlay`, read-write) unioned via
overlayfs. Creating/editing a file copies it up to `/overlay`: persists across reboots, never alters the
signed image, reversible by deleting the overlay copy, and potentially reverted by a firmware update.
