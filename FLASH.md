# Flashing CrossPoint on 阅星瞳 X4 (ESP32-C3, 16MB)

Built with PlatformIO env **`default`** (`pio run -e default`).

Board in `platformio.ini`: `esp32-c3-devkitm-1`, overridden to 16MB flash, DIO.
PlatformIO reported: `ESP32C3 160MHz, 320KB RAM, 16MB Flash`.

Checked against the sdkconfig this build actually compiled
(`framework-arduinoespressif32-libs/esp32c3/dio_qspi/include/sdkconfig.h`,
included from `.pio/build/default/src/main.cpp.d`):

| Item | Value | Where |
| --- | --- | --- |
| MCU | ESP32-C3 | image header chip id 5 |
| CPU | 160 MHz | `CONFIG_ESP_DEFAULT_CPU_FREQ_MHZ 160` |
| Crystal | 40 MHz | `CONFIG_XTAL_FREQ 40` |
| PSRAM | none | no `CONFIG_SPIRAM*` in that sdkconfig |
| Flash size | 16MB | image header (`--flash-size 16MB` at elf2image) |
| Flash mode | DIO | `CONFIG_ESPTOOLPY_FLASHMODE "dio"` and `board_build.flash_mode` |
| Flash freq | 80 MHz | `CONFIG_ESPTOOLPY_FLASHFREQ "80m"` |
| App slot | `app0` @ `0x10000`, size `0x640000` | `partitions.csv` |

`esptool image-info` (esptool 5.1.2) on the built images: ESP32-C3, flash 16MB, DIO, 80m, checksum valid.

SRAM on the chip is 400KB. The board definition's application RAM budget is 327680 bytes (what PlatformIO prints as 320KB). This image used 98260 bytes of that (30.0%). App flash used 5909115 of 6553600 bytes (90.2%).

## USB serial page-turn

The PC companion talks to the USB Serial/JTAG CDC port (`ARDUINO_USB_MODE=1`, `ARDUINO_USB_CDC_ON_BOOT=1`).

- **Baud: 115200 8N1.** `Serial.begin(115200)` in `src/main.cpp`. USB CDC does not configure parity or stop bits; open the host port at 115200.
- **`n` or `N` (`0x6E` / `0x4E`)**: next page.
- **`p` or `P` (`0x70` / `0x50`)**: previous page.
- A trailing `\r` or `\n` is ignored, so both a raw byte and `n\n` work.
- Repeats inside **250 ms** are consumed and dropped, including a burst sitting in the RX buffer.
- `n` / `p` call `HalGPIO::injectButtonPress()` on the side button `MappedInputManager` uses for page forward / page back (honors the side-button swap setting). That is the same edge path (`wasPressed` / `wasReleased` after `gpio.update()`) a Bluetooth HID page-turner would use. This tree did not already have `BluetoothHIDManager`.
- The `default` env also prints logs on this port (`LOG_LEVEL=2`). Ignore text lines; only the bytes above are commands. `CMD:SCREENSHOT` still works.

## write_flash

Images below are the `default` env output. `boot_app0.bin` is Arduino-ESP32's OTA select-app0 image (`framework-arduinoespressif32/tools/partitions/boot_app0.bin`), not produced by the app link.

Offsets:

| Offset | File | Why |
| --- | --- | --- |
| `0x0` | `bootloader.bin` | `CONFIG_BOOTLOADER_OFFSET_IN_FLASH 0x0` (ESP32-C3, not the ESP32's `0x1000`) |
| `0x8000` | `partitions.bin` | `CONFIG_PARTITION_TABLE_OFFSET 0x8000` |
| `0xe000` | `boot_app0.bin` | `otadata` in `partitions.csv` |
| `0x10000` | `firmware.bin` | `app0` / `board_upload.offset_address` |

esptool **5.1.2** (the tool that built these images) spells the command `write-flash`. esptool 4 used `write_flash`. Same arguments.

```bash
python -m esptool --chip esp32c3 -b 921600 \
  --before default-reset --after hard-reset \
  write-flash -z \
  --flash-mode dio --flash-freq 80m --flash-size 16MB \
  0x0     bootloader.bin \
  0x8000  partitions.bin \
  0xe000  boot_app0.bin \
  0x10000 firmware.bin
```

`921600` is `upload_speed` in `platformio.ini` (flashing baud). It is not the CDC console baud.

Do not write `app1` (`0x650000`) with this image. SPIFFS (`0xc90000`) and coredump (`0xFF0000`) are not part of this set.

SHA256:

```
6e6b0d386095f0783e9f0513ea29ea2ce413790e7f7153a06a2b04e2b5f59fb6  bootloader.bin
bd0f7954aca2ef7d925ee21aaa1f3dc8822d1d6ce5cbbd26a135e5886bfff6ce  partitions.bin
f94c5d786a7a8fab06ac5d10e33bf37711a6697636dc037559ea19cc410a17f0  boot_app0.bin
8f488ae7b150bde7231d21edd2ab46f18d7f6f1e05d5b3d6f7475427f993f149  firmware.bin
```

## Check on device

1. Flash, then open the USB CDC port at 115200.
2. Open a book. Send `n`, then `p`.
3. With this `default` build, the log line is `DBG [USB] Page turn next` or `prev`.
4. The 10-second `MEM` log should not drop across these commands: the path does not allocate.
5. Physical page buttons still turn pages. A burst of `n` bytes inside 250 ms turns one page.
