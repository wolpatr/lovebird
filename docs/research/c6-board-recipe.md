# MicroPython bring-up recipe: Waveshare ESP32-C6-Touch-LCD-1.28 (Windows 11)

Resolution doc for research ticket #15. All facts below were verified against primary
sources on 2026-11-02 (the board's schematic PDF, Waveshare's HARDWARE_REFERENCE.md,
Waveshare's board repo source code, the MicroPython v1.29.0 source tree and the
shipped C6 firmware image and .map file, and the ESP32-C6 datasheet). Sources are
cited inline. Anything that could not be established from a primary source is
called out explicitly (section 10).

## 0. Corrections to the ticket's premise (read first)

The S3 recipe (branch research/board-recipe) is the template, but four premises do
not carry over. All were verified against the C6 board's own schematic
(github.com/waveshareteam/ESP32-C6-Touch-LCD-1.28, hardware/schematics/ESP32-C6-Touch-LCD-1.28.pdf)
and Waveshare's machine-readable hardware reference (HARDWARE_REFERENCE.md in the
same repo, which Waveshare states is cross-checked against that schematic).

1. **There is NO USB-UART bridge chip.** Unlike the S3 board's CH343P, the Type-C
   connector wires straight to the ESP32-C6's native USB Serial/JTAG peripheral on
   GPIO12 (D-) and GPIO13 (D+). The REPL appears on Windows 11 as a native USB CDC
   COM port ("USB Serial Device", Espressif VID 303A:1001) using the inbox Windows
   CDC driver. No CH34x driver install, ever. UART0 (TXD=GPIO16, RXD=GPIO17) also
   mirrors the REPL at 115200, and both pins are on the expansion header.
2. **There is NO Waveshare MicroPython image and NO demo zip for the C6 board.**
   Waveshare's C6 resources live in the GitHub repo
   github.com/waveshareteam/ESP32-C6-Touch-LCD-1.28 (full tree scanned, 2759 files):
   Arduino v3.2.1 examples, ESP-IDF v5.5.2 examples, and
   firmware/ESP32-C6-Touch-LCD-1.28_factoryOnly.bin, which is an ESP-IDF binary
   (it contains "Starting ESP32-C6-Touch-LCD-1.28 factory UI", no MicroPython
   strings). The S3-style URL files.waveshare.com/wiki/ESP32-C6-Touch-LCD-1.28/
   ESP32-C6-Touch-LCD-1.28-Demo.zip returns 404. So the S3 recipe's "flash the
   Waveshare demo image as a zero-effort smoke test" fallback does not exist here.
3. **The charging chip is the ETA6098, not the ETA6096.** The schematic shows
   U2 = ETA6098 (with ISET table 866K/360K/160K/150K/82K/66K for
   0.2/0.5/1/1.2/2/2.5A; R5 = 360K installed, so 0.5A charge current). Both the
   English and Chinese wikis say "ETA6096 High-efficiency lithium battery charging
   chip". The schematic is authoritative for the board; the wiki text disagrees
   with Waveshare's own schematic. Nothing in the recipe depends on which exact
   ETA part it is, but vendor docs disagree and this should be settled by reading
   the chip marking when the boards arrive.
4. **There is exactly ONE ESP32_GENERIC_C6 firmware variant, not flash-size
   variants.** The download page ships a single .bin per release; the build reads
   the physical flash size at boot and auto-creates the filesystem partition to
   match (verified in v1.29.0 source, ports/esp32/main.c boardctrl_startup: "If
   there is no filesystem partition (no vfs or ffat), add a vfs partition that
   extends from the end of the application partition up to the end of flash").
   On this 16MB board that means an about 14MiB littlefs2 filesystem with zero
   extra work. There is no 4/8/16MB decision to make (unlike the S3 and the
   gc9a01_mpy prebuilts).
5. **The battery connector housing is ambiguous.** The wiki says "MX1.25 2P";
   the schematic labels P1 "GH1.25 connector 2P" (GH1.25连接器2P). Both are 1.25mm
   pitch 2-pin, but GH and MX housings are different families (the S3 board used
   MX1.25). Verify with calipers on arrival before ordering battery plugs.

## 1. Firmware: which MicroPython build to flash

### Recommended: official ESP32_GENERIC_C6 v1.29.0

- Download page: https://www.micropython.org/download/ESP32_GENERIC_C6/
- Latest stable at time of research: **v1.29.0, dated 2026-08-24**, file
  `ESP32_GENERIC_C6-20260824-v1.29.0.bin`
  (direct: https://www.micropython.org/resources/firmware/ESP32_GENERIC_C6-20260824-v1.29.0.bin,
  1,967,664 bytes).
- Scanning the actual binary confirms: "MicroPython-1.29.0-riscv-IDFv5.5.2-with-newlib4.3.0",
  "MPY version : v1.29.0 on 2026-08-24", "Machine : ESP32C6 module with ESP32-C6",
  and an ESP image header with chip id 0x0D (ESP32-C6). Built against ESP-IDF v5.5.2.
- Board list check: micropython's ESP32 port has exactly four C6 boards
  (ESP32_GENERIC_C6, M5STACK_NANOC6, SEEED_XIAO_ESP32C6, UM_TINYC6). No Waveshare
  C6 board exists, so ESP32_GENERIC_C6 is the correct official build for this board.
- Partition layout baked into the image (read from the binary at 0x8000, matches
  ports/esp32/partitions-4MiBplus.csv): nvs 0x9000 (24K), phy_init 0xF000 (4K),
  factory app 0x10000 size 0x1F0000 (1984K). The app portion (1,902,128 bytes per
  the .app-bin) fits the factory partition. Filesystem: auto-registered "vfs"
  partition from 0x200000 to the detected flash end => about 14MiB littlefs2 on
  this 16MB board (mechanism verified in main.c, see section 0 item 4).
- Build contents verified from the shipped .map file
  (ESP32_GENERIC_C6-20260824-v1.29.0.meta.zip contains the .elf and .map):
  - **USB Serial/JTAG console compiled in** (esp_driver_usb_serial_jtag and
    MicroPython's own usb_serial_jtag.c objects present). No TinyUSB/USBDevice
    (0 hits): the C6 has no native-USB-OTG, so REPL = USB Serial/JTAG + UART0.
  - **UART REPL enabled** (ports/esp32/boards/ESP32_GENERIC_C6/mpconfigboard.h:
    `#define MICROPY_HW_ENABLE_UART_REPL (1)`).
  - **RV32 native emitter enabled**: asm_rv32/emit_native_rv32 symbols present,
    so `@micropython.viper` and `@micropython.native` work. sdkconfig.c6 sets
    `CONFIG_ESP_SYSTEM_HW_STACK_GUARD=n`, which is the switch that keeps the
    native emitter available on RISC-V.
  - **`.mpy` import supported** (`MICROPY_PERSISTENT_CODE_LOAD (1)` in
    mpconfigport.h). mpy-cross v1.29.0 supports the C6 target via
    `mpy-cross -march=rv32imc` (verified in mpy-cross/main.c).
  - **Frozen modules verified from .map symbols**: umqtt_simple, umqtt_robust,
    asyncio/uasyncio, mip, urequests, ssl, dht, ds18x20, onewire, neopixel,
    ntptime, webrepl, apa106, espnow/aioespnow, inisetup, flashbdev, _boot.
    So umqtt.simple is on-device without any install step (same as the S3).
  - machine.SDCard is compiled in (machine_sdcard.c objects in .map), see
    section 8 for the C6 caveats.
- Heap configuration (ports/esp32/mpconfigport.h, v1.29.0): initial GC heap is
  64KB for the C6, and the config comment says it "starts small but adds new heap
  areas on demand due to the settings MICROPY_GC_SPLIT_HEAP and
  MICROPY_GC_SPLIT_HEAP_AUTO ... chosen so they can grow the heap once (double it)
  and still have enough internal RAM to start WiFi and make a HTTPS request."

### No alternative image is worth flashing

- Waveshare's factory bin is ESP-IDF (section 0 item 2). There is no Waveshare
  MicroPython for the C6. Flashing the factory bin is only useful to restore the
  factory UI demo (write at 0x00 per the wiki Firmware-Flashing page).
- Preview builds (v1.30.0-preview) exist on the download page; not recommended
  for bring-up.

## 2. USB, REPL and the Windows COM port

- **Native USB Serial/JTAG**: the C6 has a fixed-function USB Serial/JTAG
  peripheral (datasheet: "USB Serial/JTAG controller", SOC_USB_SERIAL_JTAG_SUPPORTED=1)
  wired directly to the Type-C (GPIO12 = USB_D-, GPIO13 = USB_D+). It carries:
  the REPL, ROM/bootloader log output (default prints to "UART0 and USB
  Serial/JTAG controller" per datasheet section 3.3), and the ROM download path
  (esptool talks to it directly).
- **Windows 11**: the board enumerates as a CDC COM port ("USB Serial Device
  (COM_n)" in Device Manager, Ports (COM & LPT)) using the inbox Windows CDC
  driver. No vendor driver download. If nothing appears: use a data-capable
  USB-C cable (charge-only cables are the classic trap) and press RESET once.
- **REPL over USB**: baudrate is irrelevant (it is USB). mpremote works normally:
  `pip install mpremote`, then `mpremote` (auto-detects the port),
  `mpremote cp main.py :/`, `mpremote exec "..."`, `mpremote mount .` for live
  development. Thonny is the GUI fallback. WebREPL is frozen in for a network REPL.
- **UART0 fallback**: TXD=GPIO16, RXD=GPIO17 at 115200, exposed on expansion
  header pins 10 (TXD) and 9 (RXD). Useful if USB ever wedges.
- esptool can put the chip into download mode over USB Serial/JTAG automatically
  (the peripheral keeps working while the main app is broken), so the BOOT
  ritual (section 3) is a fallback, not the default.

## 3. Flashing ritual on Windows 11: exact esptool commands

Sources: https://micropython.org/download/ESP32_GENERIC_C6/ installation text,
https://docs.micropython.org/en/latest/esp32/tutorial/intro.html, esptool
releases (v5.6.0 current, 2026-10-15; v5 spells the command `write-flash`, the
legacy `write_flash` still works), Waveshare FAQ
(docs.waveshare.com/ESP32-C6-Touch-LCD-1.28/FAQ).

Install esptool (on Windows the entry point is esptool.exe):

```bash
py -3.11 -m pip install esptool
```

Find the COM port: Device Manager (`devmgmt.msc`) -> Ports (COM & LPT) -> look
for "USB Serial Device (COM_n)". Typical value: COM5.

**Erase (first flash on a board that shipped with the factory firmware):**

```bash
esptool --port COM5 erase_flash
```

(MicroPython download page: "If you are putting MicroPython on your board for
the first time then you should first erase the entire flash using:
esptool.py erase_flash".)

**Flash the firmware at offset 0:**

```bash
esptool --port COM5 --baud 460800 write-flash 0 ESP32_GENERIC_C6-20260824-v1.29.0.bin
```

(MicroPython page: "deploy the firmware to the board, starting at address 0";
"if flashing starts and then fails partway through, try removing the --baud
460800 option". Over the native USB path the baud value is nominal anyway.)

**BOOT ritual (only if esptool cannot sync, or after a bad flash):** hold BOOT,
press and release RESET, release BOOT. The Waveshare FAQ for this exact board
phrases it: "hold down the BOOT button, press RESET, then release RESET, and
finally release the BOOT button. This will put the module into download mode".

**C6 strapping notes** (ESP32-C6 datasheet v1.5, section 3, tables 3-1/3-3):
boot mode is sampled from GPIO8 + GPIO9. SPI boot mode: GPIO9 = 1 (its default,
weak pull-up). Joint download boot mode: GPIO8 = 1, GPIO9 = 0. On this board
GPIO9 is the BOOT button and GPIO8 is the shared I2C SCL, held high by onboard
10K pull-ups, so the ritual just works with nothing else attached. The other
strapping-class pins (datasheet section 2.3.4 lists GPIO4, GPIO5, GPIO8, GPIO9,
GPIO15) are all wired to peripherals here: GPIO4 = TP_INT + IMU INT2,
GPIO5 = IMU INT1, GPIO15 = TF card CS. Their strapping functions are inert with
factory eFuses (the GPIO15 JTAG-source select is ignored while
EFUSE_JTAG_SEL_ENABLE=0, the default), but the general rule stands: do not hold
GPIO4/5/8/9/15 at forced levels while pressing RESET.

**Sanity checks:**

```bash
esptool --port COM5 flash_id
```

Expect "Chip is ESP32-C6" and "Detected flash size: 16MB" (the schematic's
XM25QH128DHIQT is a 16MB NOR flash; HARDWARE_REFERENCE.md confirms).

After flashing, press RESET once. If the COM port disappears or flashing fails
repeatedly, redo the BOOT ritual. GUI alternative (wiki Firmware-Flashing page):
Espressif Flash Download Tool, download address 0x00. esptool is the scriptable
path and is what lovebird uses.

## 4. Pin map (verified three ways: schematic, HARDWARE_REFERENCE.md, vendor code)

Sources: HARDWARE_REFERENCE.md (github.com/waveshareteam/ESP32-C6-Touch-LCD-1.28,
cross-checked against the schematic by Waveshare), the schematic PDF itself
(positional text extraction), the vendor BSP header
(examples/ESP-IDF-v5.5.2/08_LVGL_Demo/components/esp32_c6_touch_lcd_1_28/include/bsp/esp32_c6_touch_lcd_1_28.h)
and the Arduino examples (GC9A01.cpp, 06_HelloWorld_GFX.ino, i2c_bsp.cpp). All
four agree on every pin below.

| GPIO | Function on this board |
|---|---|
| GPIO0 | BAT_ADC: battery voltage divider (R1 200K / 100K), ADC1_CH0 |
| GPIO1 | free, expansion header pin 5, ADC1_CH1 |
| GPIO2 | free, expansion header pin 6, ADC1_CH2 |
| GPIO3 | free, expansion header pin 7, ADC1_CH3 |
| GPIO4 | TP_INT (touch interrupt) AND IMU INT2, expansion, ADC1_CH4, strapping-class |
| GPIO5 | IMU INT1, expansion, ADC1_CH5, strapping-class |
| GPIO6 | RTC_INT (PCF85063 interrupt), expansion header pin 8, ADC1_CH6 |
| GPIO7 | LCD_BL (backlight; PWM-capable via LEDC, vendor BSP uses LEDC ch 0) |
| GPIO8 | I2C_SCL, shared bus (touch + IMU + RTC), expansion pin 11, strapping (boot mode) |
| GPIO9 | BOOT button, expansion header pin 4, strapping (boot mode) |
| GPIO10 | LCD_CS |
| GPIO11 | SPI_CLK (LCD SCK and TF card SCK share it) |
| GPIO12 | USB_D- (native USB Serial/JTAG) |
| GPIO13 | USB_D+ (native USB Serial/JTAG) |
| GPIO15 | SD_CS (TF card chip select), strapping-class (JTAG source, inert by default) |
| GPIO16 | UART0 TXD, expansion header pin 10 |
| GPIO17 | UART0 RXD, expansion header pin 9 |
| GPIO18 | I2C_SDA, shared bus (touch + IMU + RTC), expansion pin 12 |
| GPIO19 | LCD_DC |
| GPIO20 | LCD_RST |
| GPIO21 | SPI_MOSI (LCD and TF card share it) |
| GPIO22 | SPI_MISO (LCD MISO and TF card MISO share it), expansion |
| GPIO23 | TP_RST; per HARDWARE_REFERENCE also relates to a BC_State signal, "do not freely drive it as a normal GPIO" |

Notes:

- GPIO14 does not exist on the QFN40 package (datasheet table 2-10 skips it);
  GPIO24-30 are the off-package flash bus. Do not chase them.
- **SPI bus**: the C6 has one general-purpose SPI peripheral (SPI2_HOST,
  SOC_SPI_PERIPH_NUM=2, one is the flash cache). In MicroPython on the C6,
  `SPI(1)` maps to SPI2_HOST (verified in ports/esp32/machine_hw_spi.c, comment
  table: "MicroPython SPI(id=1) | ESP32C6 | SPI2"; SPI(id=2) is an error on C6).
  So: `SPI(1, sck=Pin(11), mosi=Pin(21), miso=Pin(22), ...)`.
- **I2C bus**: one hardware I2C (`I2C(0, scl=Pin(8), sda=Pin(18))`, pins go
  through the GPIO matrix, any pins work) or SoftI2C. Vendor runs the bus at
  400kHz. Devices on the shared bus (HARDWARE_REFERENCE section 4):
  - CST816S touch, 7-bit 0x15, RST=GPIO23, INT=GPIO4
  - QMI8658 IMU, 7-bit 0x6B (their C driver probes 0x6A then 0x6B), INT1=GPIO5, INT2=GPIO4
  - PCF85063 RTC, 7-bit 0x51, INT=GPIO6
  Avoid adding external I2C devices at 0x15/0x51/0x6A/0x6B.
- **ADC**: single ADC unit, 7 channels, 12-bit: GPIO0=CH0 (battery), GPIO1=CH1,
  GPIO2=CH2, GPIO3=CH3, GPIO4=CH4, GPIO5=CH5, GPIO6=CH6 (datasheet table 2-9).
  `machine.ADC(Pin(0))` for the battery.
- **Deep sleep wake** (future battery pet): the C6 wakes from deep sleep only
  from GPIO0-7. Usable wake sources on this board: IMU INT1/INT2 (GPIO5/4) and
  RTC INT (GPIO6). The BOOT button (GPIO9) can NOT wake from deep sleep.
- **Expansion header P2** (SH1.0 12P, physical pin order extracted from the
  schematic): 1 GND, 2 3V3, 3 RESET (CHIP_PU), 4 BOOT (GPIO9), 5 IO1, 6 IO2,
  7 IO3, 8 IO6, 9 RXD (GPIO17), 10 TXD (GPIO16), 11 I2C_SCL (GPIO8),
  12 I2C_SDA (GPIO18). Free GPIOs for add-ons: IO1, IO2, IO3, IO6 (all
  ADC-capable) plus BOOT usable as a user button after boot.

## 5. Display driver decision (the C6 twist: no compiled driver exists)

### The evidence

- **russhughes/gc9a01_mpy publishes no C6 build.** Verified via the GitHub API:
  the repo's firmware/ directory ships ESP32_GENERIC, ESP32_GENERIC_C3,
  ESP32_GENERIC_S2, ESP32_GENERIC_S3, RP2040 families and more, but no
  ESP32_GENERIC_C6; the repo has zero releases and zero tags; no issue mentions
  C6. The prebuilt S3-era images identify as MicroPython 1.22.2, incompatible
  with the official C6 v1.29.0 ABI anyway.
- **Waveshare ships no MicroPython for the C6 board** (section 0 item 2).
- **GitHub search for maintained alternatives**: nothing with meaningful C6
  support beyond the pure-Python options below (nano-gui and gc9a01py). The
  UNIT-Electronics C6 MicroPython project uses an SSD1306 OLED, not GC9A01.
  The AndroidCrypto C6+GC9A01 clock is Arduino C++.

### The decision for v0: official firmware + vendored pure-Python/viper driver

- **peterhinch/micropython-nano-gui** (668 stars, actively maintained, pushed
  2026-06) has a GC9A01 driver family exactly designed for RAM-tight chips:
  `drivers/gc9a01/gc9a01.py` (4-bit indexed color framebuf, 28,800-byte buffer,
  viper-accelerated line conversion) and `drivers/gc9a01/gc9a01_8_bit.py`
  (8-bit rrrgggbb, 57,600-byte buffer). Both subclass framebuf.FrameBuffer and
  need nothing the official C6 build lacks (framebuf + the viper emitter, both
  verified present, section 1). MIT licensed, and its setup example is literally
  the Waveshare round touch board family.
- **russhughes/gc9a01py** (62 stars, last pushed 2021) is the plain-Python
  fallback: copy one .py to the board, zero build steps, slowest full-frame
  rate. Its GC9A01 init sequence is the standard one.
- **Waveshare's own C6 Arduino demo carries the exact init sequence for this
  panel** (examples/Arduino-v3.2.1/07_LVGL_Demo/GC9A01.cpp), including the
  reset timing (RST high 200ms, low 200ms, high 200ms, then init, SLPOUT wait
  120ms, DISPON wait 20ms), MADCTL 0x36=0x08, COLMOD 0x3A=0x05 (RGB565), and a
  79-entry register program. That file is the authoritative init source for the
  vendored driver; it is reproduced in section 9.
- **RGB565 byte order gotcha**: the GC9A01 samples MSB first; framebuf.RGB565
  on the ESP32 is little-endian, so raw framebuf bytes must be swapped on the
  way out. nano-gui handles this in its line conversion; Waveshare's LVGL
  config sets LV_COLOR_16_SWAP=y for the same reason. Any custom driver must
  handle it or colors come out wrong.

### RAM budget (the C6's first-class constraint)

No PSRAM: 512KB HP SRAM total, DRAM segment 0x6E610 = 452,304 bytes (verified
in the shipped .map), of which static .dram0.bss (32.7K) + .dram0.data (13.1K)
take about 46KB, FreeRTOS/IDF/WiFi/BLE take the rest, and MicroPython's GC heap
starts at 64KB and auto-grows. Full framebuffer options for 240x240:

| Framebuffer strategy | Bytes | Verdict |
|---|---|---|
| Full RGB565 (framebuf.RGB565) | 115,200 (112.5KB) | fits after WiFi in the best case, but eats most of the heap; avoid |
| 8-bit rrrgggbb (nano-gui 8-bit) | 57,600 (56.25KB) | good balance for a pet face |
| 4-bit indexed (nano-gui 16-bit-color LUT) | 28,800 (28.125KB) | cheapest, 16 simultaneous colors |
| Partial line buffer, dirty-rect writes (vendor LVGL uses 60 lines) | 28,800 per buffer | what Waveshare/Espressif themselves do on this board |

Recommendation: 4-bit or 8-bit indexed framebuffer for the face, or dirty-rect
partial writes. Keep the saved heap for WiFi + TLS (umqtt over TLS needs tens of
KB transient during the handshake).

### SPI clock and frame-rate expectations (estimates, clearly labeled)

- Waveshare's own C6 demos request an 80MHz pixel clock on these exact pins
  (`LCD_PIXEL_CLOCK_HZ = 80 * 1000 * 1000` in GC9A01.cpp; the BSP likewise), and
  the shipped factory firmware demonstrably runs a smooth LVGL UI, so 80MHz is
  the vendor-proven starting point. The GC9A01A datasheet allows scl up to
  100MHz (per nano-gui's citation of datasheet 7.3.4). Whether the C6's
  GPIO-matrix-routed SPI sustains the full 80MHz or the effective clock is
  lower on these non-IOMUX pins is NOT verifiable without hardware: request
  80MHz in `SPI(1, baudrate=80_000_000)`, and if a full-frame blit is unstable
  or slower than expected, drop to 40MHz and measure.
- Theoretical full-frame push time: 115,200 bytes = 23.1ms at 40MHz (43fps
  ceiling), 11.5ms at 80MHz (87fps ceiling).
- Driver-path expectations (ESTIMATES, to be measured on hardware, none of
  these are primary-source facts):
  - vendored viper/framebuf driver (nano-gui pattern): roughly 10-25fps
    full-screen refresh; small dirty-rect sprite updates far faster.
  - plain pure-Python (gc9a01py-style, no viper): roughly 2-5fps full-screen;
    fine for a mostly-static pet face with small sprite animations.
  - compiled gc9a01 C module (if built for C6): near the SPI ceiling,
    roughly 40-80fps full-screen.
- For a tamagotchi UI (small face sprites, text, occasional full redraw), all
  three paths are viable; pick the vendored pure path for v0 and only pay the
  C-build cost if measurements demand it.

### The upgrade path: building gc9a01_mpy from source for the C6

Realistic effort assessment (all steps verified to exist, effort is an estimate):
install WSL2 (the MicroPython ports/esp32/README.md explicitly recommends WSL
as the most efficient Windows toolchain path), install ESP-IDF v5.5.2 inside
it, clone micropython v1.29.0, add gc9a01_mpy's C sources as a user C module
(the repo documents a build against a micropython fork; the module is plain
C against the MicroPython C API), build `make BOARD=ESP32_GENERIC_C6`, flash
the resulting image. First run: half a day to a day. Ongoing cost: the module
ABI is pinned to the MicroPython version, so every firmware upgrade needs a
rebuild. mpy-cross rv32imc does NOT help here (a C module cannot be shipped as
.mpy). Do this only if the pure-Python path measurably fails the pet's
animation budget.

## 6. Touch driver: CST816S

- Same chip as the S3 board: Hynitron CST816S, 7-bit address 0x15, on this
  board SDA=GPIO18, SCL=GPIO8, RST=GPIO23, INT=GPIO4 (HARDWARE_REFERENCE
  section 4; matches the schematic and the vendor BSP defines
  BSP_LCD_TOUCH_RST = GPIO_NUM_23, BSP_LCD_TOUCH_INT = GPIO_NUM_4).
- **Vendor the same cst816.py the S3 recipe uses** (from the S3 demo zip,
  ESP32-S3-Touch-LCD-1.28-Demo/MicroPython/cst816.py, MIT, derived from
  NeoStormer/CircuitPython_CST816) and change only the pins:
  `I2C(0, scl=Pin(8), sda=Pin(18), freq=400000)`, reset on `Pin(23)`,
  interrupt (if used) on `Pin(4)`. The register map is unchanged:
  gesture 0x01, finger count 0x02, X/Y 0x03-0x06, chip ID 0xA7 (expects 0xB5),
  gestures up=0x01 down=0x02 left=0x03 right=0x04 click=0x05
  double=0x0B long=0x0C (verified against both the vendored driver and the
  C6 repo's Arduino CST816S.h by Felix Biego, MIT).
- **Shared-interrupt design note**: GPIO4 is TP_INT and IMU INT2 at the same
  time (HARDWARE_REFERENCE: "Touch and IMU_INT2 share this GPIO; avoid
  interrupt conflicts"). Prefer polling touch from the main loop (the vendored
  driver polls) and use the IMU's INT1 (GPIO5) only, leaving INT2 unwired in
  firmware.
- The C6 repo's own C reference for the touch chip is
  examples/Arduino-v3.2.1/07_LVGL_Demo/CST816S.cpp (Biego driver) if deeper
  register semantics are ever needed. Waveshare's CST816S datasheet and
  register description PDFs are linked from the wiki Resources page
  (files.waveshare.com/wiki/common/CST816S_Datasheet_EN.pdf,
  CST816S_register_declaration.pdf).

## 7. IMU driver: QMI8658

- QMI8658 6-axis (3-axis gyro + 3-axis accel), shared I2C bus, 7-bit address:
  HARDWARE_REFERENCE says 0x6B; Waveshare's own C driver defines both
  QMI8658_SLAVE_ADDR_L = 0x6A and _H = 0x6B and probes both in
  qmi8658_get_id() (examples/Arduino-v3.2.1/02_I2C_QMI8658/qmi8658c.h/.cpp).
  Scan the bus on first boot and take what answers (expect 0x6B per the
  reference doc).
- INT1=GPIO5, INT2=GPIO4 (GPIO4 shared with touch INT, see section 6).
- Existing MicroPython drivers (small, but real and MIT-style permissive):
  echo-lalia/qmi8658-micropython (6 stars, 2024), sebromero/micropython-qmi8658c
  (2025-12), mattytrentini/micropython-qmi8658 (2026-09). None is
  board-specific; all need the C6 pins passed in.
- The register-level reference is Waveshare's own qmi8658c.cpp (28KB, full init
  plus attitude conversion) in the C6 repo. For lovebird: read raw accel/gyro
  at 20-50Hz and detect shake/tilt in Python; port only the init and raw-read
  registers from the C driver (whoami, ctrl registers, data registers), not
  the whole attitude stack.

## 8. The board's extras (RTC, TF, battery, charging, buttons)

- **RTC: PCF85063ATL** at 0x51, INT=GPIO6, 32.768kHz crystal (Y2). Keeps time
  across main-power-off when the SH1.0 2P RTC battery header (J5) carries a
  rechargeable cell (wiki: "Supports connecting a rechargeable RTC battery";
  the schematic marks the RTC section "RTC(可选)" = optional). MicroPython's
  internal machine.RTC does NOT survive power-off, so for a pet that remembers
  real time, drive the PCF85063 over I2C. Register map from Waveshare's
  pcf85063a.h: control 0x00-0x03, seconds 0x04, minutes 0x05, hours 0x06,
  day 0x07, weekday 0x08, month 0x09, year 0x0A with "real year = 1970 +
  register year". Existing MicroPython library:
  SolderedElectronics/Soldered-PCF85063A-MicroPython-Library (MIT). RTC INT on
  GPIO6 can also deep-sleep-wake the pet (GPIO0-7 wake restriction, section 4).
- **TF card slot**: SPI mode on the shared SPI2 bus: SCK=GPIO11, MOSI=GPIO21,
  MISO=GPIO22, CS=GPIO15 (SD_CS). machine.SDCard is compiled into the official
  C6 build (verified in .map). The C6 has no SDMMC host, so SPI mode is the
  only mode; the ESP32 port's slot=2 is the SPI slot, and the C6 has NO
  default pin assignment (ports/esp32/machine_sdcard.h: "Other chips have no
  historical default; a board or the runtime must assign"), so bring-up must
  pass the pins explicitly. The exact v1.29.0 constructor kwarg set is listed
  in the machine.SDCard docs and should be confirmed there at bring-up time
  (called out in section 10). Use case for lovebird: sprite/asset storage,
  cheaper than burning flash.
- **Battery header + charging**: 2P 1.25mm header (P1; housing family ambiguous,
  section 0 item 5). Charging IC ETA6098 (schematic; wiki says 6096) with
  charge current set by ISET R5=360K => 0.5A (schematic ISET table:
  866K/360K/160K/150K/82K/66K = 0.2/0.5/1/1.2/2/2.5A). Charge indicator LED D2:
  per the wiki, on while charging, off when full, undefined with no battery.
  Power path: AO3401 P-FET + MBR230 Schottky ORing USB and battery; 3V3 buck
  is the MP1605GTF-Z.
- **Battery voltage readout**: divider R1 200K / 100K into GPIO0 (BAT_ADC),
  same 3:1 topology as the S3 board. Formula (12-bit-equivalent, read via
  read_u16): `vbat = adc.read_u16() * 3.3 * 3 / 65535`, calibrate against a
  multimeter. The S3 wiki's formula had the same shape
  (3.3 / (1<<12) * 3 * AD_Value).
- **Vibration motor: NOT present.** The C6 board has no MOS-controlled motor
  pads (the S3 board had MOSFET1_CS/MOSFET2_CS on GPIO4/5; the C6 schematic
  has no such nets). For hug feedback later: external MOSFET (about 2 euro)
  wired to a spare expansion GPIO (IO1/IO2/IO3/IO6 are free and even
  ADC-capable). Same conclusion as the S3 extras plan, but off-board.
- **Buttons**: BOOT (GPIO9) is usable as a normal user button after boot (it
  is expansion header pin 4); the only caveat is the strapping one: holding it
  during reset enters download mode. RESET is wired to expansion pin 3. So the
  pet gets one free physical button with zero hardware work.

## 9. Known-good skeleton main.py (assembled from verified pieces)

Everything below is constructed from verified primary-source pieces: the pin
table (section 4), Waveshare's own GC9A01 init for this exact panel (GC9A01.cpp),
the vendored cst816.py port (section 6), and the battery divider (section 8).
It is NOT yet hardware-tested (the boards are 15-20 days out); the build ticket
vendors the driver file and runs it for real.

```python
# lovebird bring-up sketch: Waveshare ESP32-C6-Touch-LCD-1.28
# MicroPython ESP32_GENERIC_C6 v1.29.0, pins per docs/research/c6-board-recipe.md
import time
from machine import Pin, SPI, I2C, ADC

# ---- display: GC9A01 on SPI2 (= machine.SPI(1) on the C6) ----
# cs=10 dc=19 rst=20 bl=7, sck=11 mosi=21 (miso=22 optional for LCD)
# vendored driver is a framebuf.FrameBuffer subclass (nano-gui pattern),
# byte-swapping RGB565 on the way out (GC9A01 samples MSB first)
spi = SPI(1, baudrate=80_000_000, polarity=0, phase=0,
          sck=Pin(11), mosi=Pin(21), miso=Pin(22))
# tft = gc9a01_fb.GC9A01(spi, cs=Pin(10, Pin.OUT, value=1),
#                        dc=Pin(19, Pin.OUT, value=0),
#                        rst=Pin(20, Pin.OUT, value=1))
# backlight on GPIO7: plain on/off, or machine.PWM(Pin(7)) for dimming
bl = Pin(7, Pin.OUT, value=1)

# The vendored driver must send Waveshare's own init sequence for this panel
# (copied verbatim from their C6 Arduino GC9A01.cpp, do not trim entries):
INIT_SEQUENCE = [
    (0xEF, b""), (0xEB, b"\x14"), (0xFE, b""), (0xEF, b""),
    (0xEB, b"\x14"), (0x84, b"\x40"), (0x85, b"\xff"), (0x86, b"\xff"),
    (0x87, b"\xff"), (0x88, b"\x0a"), (0x89, b"\x21"), (0x8A, b"\x00"),
    (0x8B, b"\x80"), (0x8C, b"\x01"), (0x8D, b"\x01"), (0x8E, b"\xff"),
    (0x8F, b"\xff"), (0xB6, b"\x00\x20"), (0x36, b"\x08"),   # MADCTL portrait
    (0x3A, b"\x05"),                                          # COLMOD RGB565
    (0x90, b"\x08\x08\x08\x08"), (0xBD, b"\x06"), (0xBC, b"\x00"),
    (0xFF, b"\x60\x01\x04"), (0xC3, b"\x13"), (0xC4, b"\x13"),
    (0xC9, b"\x22"), (0xBE, b"\x11"), (0xE1, b"\x10\x0e"),
    (0xDF, b"\x21\x0c\x02"),
    (0xF0, b"\x45\x09\x08\x08\x26\x2a"), (0xF1, b"\x43\x70\x72\x36\x37\x6f"),
    (0xF2, b"\x45\x09\x08\x08\x26\x2a"), (0xF3, b"\x43\x70\x72\x36\x37\x6f"),
    (0xED, b"\x1b\x0b"), (0xAE, b"\x77"), (0xCD, b"\x63"),
    (0x70, b"\x07\x07\x04\x0e\x0f\x09\x07\x08\x03"), (0xE8, b"\x34"),
    (0x62, b"\x18\x0d\x71\xed\x70\x70\x18\x0f\x71\xef\x70\x70"),
    (0x63, b"\x18\x11\x71\xf1\x70\x70\x18\x13\x71\xf3\x70\x70"),
    (0x64, b"\x28\x29\xf1\x01\xf1\x00\x07"),
    (0x66, b"\x3c\x00\xcd\x67\x45\x45\x10\x00\x00\x00"),
    (0x67, b"\x00\x3c\x00\x00\x00\x01\x54\x10\x32\x98"),
    (0x74, b"\x10\x85\x80\x00\x00\x4e\x00"), (0x98, b"\x3e\x07"),
    (0x35, b""), (0x21, b""), (0x11, b""),   # SLPOUT
    # wait 120 ms here
    (0x29, b""),                              # DISPON
    # wait 20 ms here
]
# reset timing per Waveshare GC9A01.cpp: RST high 200 ms, low 200 ms, high 200 ms

# ---- touch: CST816S on the shared I2C bus ----
# vendored cst816.py (S3 demo, MIT, NeoStormer) with C6 pins:
#   I2C(0, scl=Pin(8), sda=Pin(18), freq=400000), rst=Pin(23, Pin.OUT)
# touch.who_am_i() reads reg 0xA7 and must return 0xB5
# touch.get_point() -> x/y, touch.get_gesture() -> 1..5, 0x0B, 0x0C
# GPIO4 is TP_INT *and* IMU INT2: poll touch, do not fight over the IRQ

# ---- battery: divider 200K/100K on GPIO0 (ADC1_CH0) ----
adc = ADC(Pin(0))
raw = adc.read_u16()
vbat = raw * 3.3 * 3 / 65535   # calibrate against a multimeter

# ---- first-boot measurements to record (fills section 10's gaps) ----
import gc
print("free heap:", gc.mem_free())      # before WiFi
# import network; ... connect ...
# print("free heap after wifi:", gc.mem_free())
```

## 10. Facts that could NOT be established from primary sources

Stated explicitly per the ticket:

- **Exact free heap at boot.** Not measurable without the hardware. What IS
  verified: 512KB HP SRAM, a 452,304-byte DRAM segment, about 46KB static DRAM
  in the firmware, a 64KB initial GC heap that auto-grows (split heap), and
  the config comment that it is sized to double and still start WiFi + HTTPS.
  Expectation (an estimate, not a fact): order 150-250KB total Python heap.
  First-boot action: print gc.mem_free() before and after WiFi connect and
  write the numbers into the firmware skeleton's README.
- **Achieved SPI clock at 80MHz through the GPIO matrix on these pins.** The
  vendor requests 80MHz and ships it in the factory firmware, but no primary
  source pins the C6's GPIO-matrix SPI ceiling, and MicroPython does not report
  the actual achieved clock back. Measure full-frame blit times on hardware.
- **Real frame rates per driver path.** Section 5's fps numbers are reasoned
  estimates from bus math and driver architecture, not measurements.
- **Battery connector housing family** (GH1.25 per schematic vs MX1.25 per
  wiki text; same 1.25mm pitch, different plug families). Calipers on arrival.
- **ETA6098 vs ETA6096.** Schematic vs wiki text disagree (section 0 item 3);
  the chip marking settles it. Electrically both are 1-cell Li chargers and
  nothing in the recipe depends on the exact part.
- **Whether charge status is readable in software.** The charge LED is driven
  by the ETA chip's STAT pin directly; HARDWARE_REFERENCE warns GPIO23 also
  relates to a "BC_State" signal, but the exact relationship is not documented
  clearly enough to rely on. Treat GPIO23 as touch-reset only.
- **machine.SDCard(slot=2) pin kwarg names on v1.29.0 for the C6.** The C code
  accepts pin overrides via a SET_CONFIG_PIN macro, but the exact constructor
  kwargs (and whether CS is passed per-slot) should be confirmed against the
  v1.29.0 machine.SDCard documentation at bring-up time; the C6 has no default
  pins, so explicit configuration is definitely required.
- **GC9A01A panel variance.** Waveshare's init is tuned for the exact panel
  they ship; if a future board revision changes the panel, the init sequence
  may need re-checking against GC9A01A.pdf (wiki Resources).

## Source index

- Waveshare board documentation (EN): https://docs.waveshare.com/ESP32-C6-Touch-LCD-1.28
  (subpages: Resources-And-Documents, Firmware-Flashing, Development-Environment-Setup-Arduino,
  Development-Environment-Setup-ESP-IDF, FAQ)
- Waveshare board documentation (CN mirror): https://docs.waveshare.net/ESP32-C6-Touch-LCD-1.28/
- Waveshare product page: https://www.waveshare.com/esp32-c6-touch-lcd-1.28.htm
- Waveshare board repo (examples, schematic, hardware reference, factory firmware):
  https://github.com/waveshareteam/ESP32-C6-Touch-LCD-1.28
  - HARDWARE_REFERENCE.md (cross-checked pin/GPIO/I2C tables)
  - hardware/schematics/ESP32-C6-Touch-LCD-1.28.pdf (schematic, single page)
  - examples/Arduino-v3.2.1/07_LVGL_Demo/GC9A01.cpp (display init, 80MHz clock, reset timing)
  - examples/Arduino-v3.2.1/07_LVGL_Demo/CST816S.cpp/.h (touch register reference, Felix Biego MIT)
  - examples/Arduino-v3.2.1/02_I2C_QMI8658/qmi8658c.h/.cpp (IMU registers, address probe 0x6A/0x6B)
  - examples/Arduino-v3.2.1/01_I2C_PCF85063/pcf85063a.h (RTC register map)
  - examples/Arduino-v3.2.1/02_I2C_QMI8658/i2c_bsp.cpp (I2C pins GPIO8/GPIO18, 400kHz)
  - examples/ESP-IDF-v5.5.2/08_LVGL_Demo/components/esp32_c6_touch_lcd_1_28/ (BSP pin defines)
  - firmware/ESP32-C6-Touch-LCD-1.28_factoryOnly.bin (ESP-IDF factory UI, not MicroPython)
- MicroPython ESP32_GENERIC_C6 downloads: https://www.micropython.org/download/ESP32_GENERIC_C6/
- MicroPython v1.29.0 source (esp32 port):
  https://github.com/micropython/micropython/tree/v1.29.0/ports/esp32
  (main.c auto-vfs, mpconfigport.h heap/emitter/persistent-code config,
  boards/ESP32_GENERIC_C6/mpconfigboard.h, boards/sdkconfig.c6, machine_hw_spi.c
  SPI id mapping, usb_serial_jtag.c, flashbdev.py, inisetup.py,
  partitions-4MiBplus.csv, machine_sdcard.c/.h, README.md build instructions)
- MicroPython v1.29.0 C6 firmware artifacts (scanned: binary strings, image
  header, partition table, and the .elf/.map from the .meta.zip):
  https://www.micropython.org/resources/firmware/ESP32_GENERIC_C6-20260824-v1.29.0.bin
- mpy-cross rv32imc support: https://github.com/micropython/micropython/blob/v1.29.0/mpy-cross/main.c
- MicroPython ESP32 tutorial: https://docs.micropython.org/en/latest/esp32/tutorial/intro.html
- esptool releases: https://github.com/espressif/esptool/releases (v5.6.0, 2026-10-15)
- ESP32-C6 datasheet v1.5: https://documentation.espressif.com/esp32-c6_datasheet_en.pdf
  (boot mode table 3-3, strapping pins sections 2.3.4/3, pin tables 2-9/2-10,
  ADC channels, USB Serial/JTAG)
- ESP-IDF v5.5.2 C6 SoC capabilities:
  https://github.com/espressif/esp-idf/blob/v5.5.2/components/soc/esp32c6/include/soc/soc_caps.h
- GC9A01 compiled driver (no C6 build): https://github.com/russhughes/gc9a01_mpy
- GC9A01 pure-Python driver (2021): https://github.com/russhughes/gc9a01py
- nano-gui GC9A01 drivers (vendoring basis): https://github.com/peterhinch/micropython-nano-gui
  (drivers/gc9a01/gc9a01.py, drivers/gc9a01/gc9a01_8_bit.py, setup_examples/gc9a01_pico.py)
- CST816S upstream of the vendored driver: https://github.com/NeoStormer/CircuitPython_CST816
- CST816S datasheet + register description (wiki links):
  https://files.waveshare.com/wiki/common/CST816S_Datasheet_EN.pdf,
  https://files.waveshare.com/wiki/common/CST816S_register_declaration.pdf
- GC9A01A datasheet (wiki link): https://files.waveshare.com/wiki/common/GC9A01A.pdf
- S3 demo zip (source of the vendored cst816.py and the S3 recipe it came from):
  https://files.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28/ESP32-S3-Touch-LCD-1.28-Demo.zip
  and branch research/board-recipe, docs/research/board-recipe.md in this repo
- QMI8658 MicroPython drivers: https://github.com/echo-lalia/qmi8658-micropython,
  https://github.com/sebromero/micropython-qmi8658c,
  https://github.com/mattytrentini/micropython-qmi8658
- PCF85063A MicroPython library: https://github.com/SolderedElectronics/Soldered-PCF85063A-MicroPython-Library