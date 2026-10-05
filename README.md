# Pager Field Notes

Unofficial, community field notes for the **Hak5 WiFi Pineapple Pager** — the dev- and
reverse-engineering-level details that make it easier to build themes and payloads (and to let AI
assistants help), written as plain Markdown you can read, clone, or feed to a tool.

> **Unofficial & not affiliated with Hak5.** These are hands-on findings from using and testing a
> retail device, in our own words. They complement — they do not copy or replace — the official docs
> at `docs.hak5.org/wifi-pineapple-pager`. No Hak5 firmware images, binaries, or documentation text
> are included or redistributed here.

## How to use this

- **Just read it.** Every page is Markdown; browse on GitHub or clone the repo.
- **Feed it to an AI.** Point your assistant at the repo (or paste a page) — that's the whole idea.
- **As a GitBook.** The repo is GitBook-compatible: `SUMMARY.md` is the table of contents and
  `.gitbook.yaml` the config, so it can be synced to a GitBook space, but nothing requires GitBook.

## Confidence & versions

Firmware moves; treat these as point-in-time notes. Most were **verified on hardware**; a few are
**inferred** and labelled as such. Where a detail is version-specific it names the firmware. Current
reference point: firmware **1.1.3** (`DISTRIB_RELEASE 24.10.1`), theme framework **0.8**.

## Contents

- **Device** — [hardware & OS](device/hardware.md), [display & input](device/display-and-input.md),
  [local API & helpers](device/local-api.md), [sensors & sysfs](device/sensors.md), [USB](device/usb.md)
- **Themes** — [format](themes/format.md), [firmware notes & gotchas](themes/firmware-notes.md)
- **Payloads** — [basics](payloads/basics.md), [building native/Go tools](payloads/building.md),
  [proven patterns](payloads/patterns.md)
- **Firmware findings** — [confirmed bugs & launch internals](firmware/findings.md)

## Contributing

Issues/PRs with corrections or additions are welcome — please note the **firmware version** you
observed something on, and whether it's **verified** or **inferred**. Keep it to original,
own-words findings; do not paste Hak5's docs, firmware, or binaries.

## License

Prose is **CC BY 4.0**; code snippets in these notes are **MIT** — use freely with attribution.
This covers only the notes themselves, not any Hak5 property.

## Safety

This documents a security-testing tool you own, for building themes/payloads and understanding the
device — the same kind of material Hak5 publishes. Use it legally and only on systems you're
authorized to test.
