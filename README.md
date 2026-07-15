# claude-usage-fw-jc4880p4

OTA release channel for the **Guition JC4880P443C_I_W (ESP32-P4)** build of the
Claude-Usage firmware.

The device's `Check for Update` / auto-OTA polls this repo's
`/releases/latest`. Each release carries an **app-only** OTA image named
`firmware-jc4880p4-<tag>.bin` (flashed to the ota_0/ota_1 partition; the
bootloader + partition table are untouched). The plain merged image is NOT
OTA-flashable.

Separate from the S3 board (`asrth/claude-usage-fw-jc3248`) and the M5
(`asrth/claude-usage-fw`) — different chip, different binary.

## Docs

- **[Updating the ESP32-C6 Wi-Fi co-processor firmware](docs/c6-wifi-firmware-update.md)**
  — required once per board: the factory C6 ships ESP-Hosted slave 2.3.0, which makes
  Wi-Fi associate-then-drop (`ASSOC_LEAVE` loop) with this firmware's 2.12.x host.
  Covers exactly which JP1 pins to tap and the esptool commands.
