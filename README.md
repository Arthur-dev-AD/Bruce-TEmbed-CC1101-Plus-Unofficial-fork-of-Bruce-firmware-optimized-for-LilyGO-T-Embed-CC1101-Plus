# Bruce for LilyGO T-Embed CC1101 Plus — Unofficial Fork

> **This is an unofficial, modified fork of Bruce firmware.**
> It is **not** affiliated with, endorsed by, or supported by the Bruce project
> or LilyGO. For the official firmware, boards and support, use
> **https://github.com/BruceDevices/firmware** and **https://bruce.computer**.

A trimmed-down, T-Embed CC1101 Plus-focused build of
[Bruce](https://github.com/BruceDevices/firmware): optimized for the LilyGO
**T-Embed CC1101 Plus**, with useless features removed and the NFC/RFID write
path fixed.

## What this fork changes

### Target device
- Preconfigured and tuned for the **LilyGO T-Embed CC1101 Plus**
  (ESP32-S3, 16 MB flash, PN532 RFID via I2C, CC1101 sub-GHz).

### Removed / disabled
- **125 kHz RFID reader removed** (`REMOVE_125KHZ_READER`): the T-Embed CC1101
  Plus has no 125 kHz front-end, so the menu entries and driver were dead weight.
- **LoRa** removed entirely: `LoRaMenu` plus the `src/modules/lora/` driver
  (`LoRaRF`) were deleted; no main-menu entry remains.
- **FM (SI4713)** removed entirely: `FMMenu` plus the `src/modules/fm/` driver
  were deleted; no main-menu entry remains.
- **GPS** removed from the main menu (`GpsMenu` deleted). The GPS module code is
  still present because it backs the wardriving features used elsewhere.
- **Ethernet** removed from the main menu (`EthernetMenu` deleted). The Ethernet
  module code is still present because the ARP scanner is used by Wi-Fi tools.
- The main menu was trimmed accordingly (`src/core/main_menu.{h,cpp}`), reducing
  flash size and boot/RAM overhead.

### NFC / RFID (PN532) fixes
- **Fixed a data-loss bug in `Erase data`**: the erase loop used to write up to
  page **129** regardless of the card's real size. On NTAG clone chips (e.g. the
  Fudan NTAG213 that ships with the T-Embed), writing past the end of memory hits
  configuration / lock areas and can **permanently write-lock the card**. The
  erase now reads the capability-container (CC) byte to detect NTAG213 / 215 / 216
  and **never writes past the last user page**.
- Locked pages (static lock bits) are now skipped during erase instead of
  aborting the whole operation.
- **Real write verification**: every write is read back and compared; a failed
  read can no longer be reported as "written" (no more fake progress).
- Added retry + **re-selection of the PICC** when a write is refused or a
  verify mismatches, which recovers sessions the PN532 would otherwise drop.
- Failure diagnostics on screen: "Waiting for tag...", "Write failed: page N"
  and "Lost tag at page N".
- Write throttling to stop hammering the PN532 (previous builds polled every
  ~1 ms while reads waited 2 s).

### Battery gauge & power-off fixes
- **Fixed the "battery shows 100% then the device won't turn off" bug.** The
  BQ27220 fuel gauge and the BQ25896 PMIC share the same I2C (`Wire`) bus. A
  failed gauge read used to return garbage (clamped to **100%**) *and* could
  wedge the bus, which silently broke `PPM.shutdown()` — the device could then
  only be recovered with a hardware reset.
- `i2cReadBytes` now validates the I2C transaction, zeroes the buffer and retries
  (up to 3×); a failed read returns a sentinel (`0xFFFF`) instead of garbage.
- `getBattery()` rejects the sentinel and impossible values (> 100) and keeps the
  **last known-good percentage** instead of jumping to 100%.
- `powerOff()` re-initialises the I2C bus before calling `PPM.shutdown()` and
  falls back to a hardware deep-sleep "off" (drops the power-hold rail, wakes on
  GPIO 6) if the PMIC does not answer — the device always powers off.

## Building

```sh
py -3.13 -m platformio run -e lilygo-t-embed-cc1101
```

The merged firmware image is produced at the repo root as
`Bruce-lilygo-t-embed-cc1101.bin`.

## Flashing

```sh
esptool.py --chip esp32s3 --port <PORT> write_flash 0x00000 Bruce-lilygo-t-embed-cc1101.bin
```

## License and attribution

This fork is distributed under the **GNU Affero General Public License v3.0 or
later (AGPL-3.0-or-later)**, the same license as the upstream Bruce project.
See [`LICENSE`](LICENSE) and [`THIRD_PARTY.md`](THIRD_PARTY.md).

- Original project: **Bruce** — <https://github.com/BruceDevices/firmware>
- Upstream copyright (C) the Bruce contributors.
- This fork contains modifications made by its author and is **not** the
  official Bruce firmware.

Because this is AGPL software, if you redistribute this firmware (source or
binary) you must keep the license and copyright notices, provide the complete
corresponding source, and clearly indicate that it is a modified version.

The name "Bruce" and any project branding remain the property of their owners
and are used here only to describe the origin of this derivative work.
