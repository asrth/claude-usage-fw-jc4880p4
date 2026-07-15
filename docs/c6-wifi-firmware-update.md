# Updating the ESP32-C6 Wi-Fi co-processor firmware (JC4880P443C_I_W)

The Guition JC4880P443 board pairs the ESP32-P4 (which runs this firmware) with an
**ESP32-C6** Wi-Fi/BT co-processor connected over ESP-Hosted (SDIO). The board **ships
with an old ESP-Hosted slave 2.3.0** on the C6, while arduino-esp32 3.3.x's host needs
**~2.12.x** — the mismatch causes the classic symptoms:

- Wi-Fi associates then immediately drops (`ASSOC_LEAVE`, reason 8), over and over
- SDIO "dropping packets" spam, restart loops, 0-for-N join attempts
- Boot log literally warns:
  `Version mismatch: Host [2.12.x] > Co-proc [2.3.0] ==> Upgrade co-proc to avoid RPC timeouts`

The fix is a one-time flash of the C6 with a current ESP-Hosted slave image, over the
C6's UART pins exposed on the **Expand-IO header JP1**. Verified on real hardware
(slave 2.12.9): joins cleanly with `assoc_leaves=0` afterwards.

---

## What you need

- A **3.3 V USB–TTL adapter** — **use FTDI (FT232) or CP2102**, must be 3.3 V logic.
  ⚠️ **Avoid WCH CH342/CH343 dual-UART adapters on macOS**: the Apple driver wedges
  the TX path on any burst larger than a few hundred bytes (sync/erase work, the
  first data block dies with `Write timeout` / "chip stopped responding").
  Deasserting DTR/RTS via an `esptool.cfg` `custom_reset_sequence = D0|R0|W0.3`
  (+ `--before default-reset`) revives small commands but bulk writes still fail —
  an FTDI adapter flashed the same board first try (1.2 MB @460800 in 17 s).
- Jumper wires
- [`esptool`](https://docs.espressif.com/projects/esptool/) on the host PC
- The slave image: **`network_adapter_esp32c6.bin`** (2.12.x) from
  <https://esphome.github.io/esp-hosted-firmware/> (pick the plain esp32c6
  network-adapter build)

## Which pins (JP1, the 26-pin Expand-IO header)

Schematic sheet 03 ("Expand IO"). The C6 signals are the **bottom four even pins** on
the right-hand column; power/ground are at the top. Verify against your board's silk —
pin 1 is the VCC3V3 corner.

| JP1 pin | Signal        | Connect to (USB-TTL)                         |
|--------:|---------------|----------------------------------------------|
| 20      | `C6_U0RXD`    | adapter **TX**                                |
| 22      | `C6_U0TXD`    | adapter **RX**                                |
| any GND (e.g. 4/18) | `GND` | adapter **GND**                       |
| 24      | `C6_IO9`      | *(optional)* adapter **DTR** — boot strap     |
| 26      | `C6_CHIP_PU`  | *(optional)* adapter **RTS** — C6 enable/reset|
| 1       | `VCC3V3`      | do **NOT** power from the adapter — power the board over its own USB |

Three wires (TX/RX/GND) are enough if your adapter's DTR/RTS auto-reset works through
the flow below; wiring DTR→IO9 + RTS→CHIP_PU gives you esptool's automatic
download-mode entry. Manual fallback: hold **IO9 low** while pulsing **CHIP_PU**
low→high, then release IO9 — that's the standard ESP32-C6 download-mode dance.

## Procedure

### 1. HALT THE P4 FIRST — this is the step everyone misses

The running P4 app owns the C6: ESP-Hosted **resets the C6 via P4 GPIO54** whenever the
link hiccups. If the P4 keeps running, it will yank the C6's reset **mid-flash** — small
writes appear to work, large ones die with "chip stopped responding".

Park the P4 in its bootloader (it stays halted because of `--after no_reset`):

```sh
esptool --chip esp32p4 -p /dev/cu.usbmodemXXXX \
        --before default_reset --after no_reset flash_id
```

(`/dev/cu.usbmodemXXXX` = the board's own USB-C port. Any harmless read-only command
works; `flash_id` is just convenient.)

### 2. Flash the C6 over the USB-TTL adapter

```sh
# C6 partition layout: ota_0 @ 0x10000 (1.5 MB), ota_1 @ 0x190000, otadata @ 0xd000
esptool --chip esp32c6 -p /dev/cu.usbserial-XXXX --baud 115200 \
        write_flash 0x10000 network_adapter_esp32c6.bin

# Make the bootloader boot ota_0 (wipe the OTA-data selector):
esptool --chip esp32c6 -p /dev/cu.usbserial-XXXX \
        erase_region 0xd000 0x2000
```

115200 baud is slow (~3 min for 1.5 MB) but rock solid over jumper wires; feel free to
try `--baud 460800`.

### 3. Reset both chips

1. Hard-reset the **C6** (pulse `C6_CHIP_PU` low→high, or power-cycle the board).
2. Reset the **P4** (press its reset / power-cycle / re-plug USB).

### 4. Verify

Watch the P4 serial log at boot. Success looks like:

- no more `Version mismatch` warning (host and slave both 2.12.x)
- Wi-Fi joins in a few seconds, `assoc_leaves=0`, and stays up

> A "host **newer** than slave" *info* line with both on 2.12.x is harmless — only the
> big 2.3.0 gap breaks association.

## Notes

- **Do not try the ESP-Hosted OTA flasher for this jump**: the esphome-built image
  fails the hosted `ota_end` validation (`OTA_VALIDATE_FAILED`, app-desc magic
  mismatch) when pushed from a 2.3.0 slave. The direct UART flash above bypasses that
  entirely.
- The board's Wi-Fi antenna, SDIO wiring (P4↔C6: CLK18/CMD19/D0-3 = 14-17, reset
  GPIO54) and everything else are untouched by this procedure — it only rewrites the
  C6's application flash.
- Full board pin map & troubleshooting:
  <https://github.com/ultramcu/guition-jc4880p443c-i-w>
