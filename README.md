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
