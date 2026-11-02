# MicroPython bring-up recipe: Waveshare ESP32-S3-Touch-LCD-1.28 (Windows 11)

Resolution doc for research ticket #4. All facts below were verified against primary
sources on 2026-11-02. Sources are cited inline. Anything that could not be
established from a primary source is called out explicitly.

## 0. Two corrections to the ticket's premise (read first)

Research on the actual board shows two premises in the ticket that do not match the
hardware. Both were verified against the official Waveshare wiki and the Waveshare
product page.

1. **Flash is 16MB, not 8MB.** The board uses a Waveshare-listed W25Q128JVSIQ
   16MB NOR flash chip, not 8MB. (Source: Waveshare wiki "Onboard Resources"
   section, https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28; product page
   "What's On Board" https://www.waveshare.com/esp32-s3-touch-lcd-1.28.htm. The
   wiki states "W25Q128JVSIQ 16MB NOR-Flash" and "external 16MB Flash and 2MB
   PSRAM".) 8MB firmware images still work (they just leave half the flash
   unused), but the correct build to pick is the 16MB one. This matters for
   `esptool flash_id` sanity checks and for picking a prebuilt gc9a01_mpy image.
2. **There is no five-way joystick on this board.** The ESP32-S3-Touch-LCD-1.28
   has only two buttons: RESET and BOOT. Input is the capacitive touch screen
   (CST816S) plus the QMI8658 six-axis IMU (3-axis accelerometer + 3-axis
   gyroscope) for motion/gesture detection. The wiki's "Project Resources"
   section and pin table make no mention of any joystick. The five-way joystick
   is a feature of a different Waveshare round board (the RP2040-LCD-1.28).
   The lovebird interaction vocabulary should therefore lean on touch and the
   IMU (e.g. shake/tilt gestures), not a joystick. If the project README or the
   wayfinder map still assumes a joystick, those docs need a correction, but
   that is outside this ticket's scope.

## 1. Firmware: which MicroPython build to flash

### Recommended: official ESP32_GENERIC_S3 v1.29.0

- Download page: https://micropython.org/download/ESP32_GENERIC_S3/
- The page states: "The following files are firmware that should work on most
  ESP32-S3-based boards with 4MiB or more of flash" and "This firmware supports
  configurations with and without SPIRAM (also known as PSRAM) and will
  auto-detect a connected SPIRAM chip at startup".
- Latest stable at time of research: **v1.29.0, dated 2026-08-24**, file
  `ESP32_GENERIC_S3-20260824-v1.29.0.bin`
  (direct: https://micropython.org/resources/firmware/ESP32_GENERIC_S3-20260824-v1.29.0.bin).
- Board list check (ticket asks): micropython.org's ESP32 port lists exactly one
  Waveshare S3 board, WAVESHARE_ESP32_S3_PICO, which is a different board. There
  is no official Waveshare ESP32-S3-Touch-LCD-1.28 entry, so ESP32_GENERIC_S3 is
  the correct official build for this board.
- USB CDC: verified by scanning the actual v1.29.0 binary: it contains
  `USBDevice`, `BUILTIN_CDC` and "Board CDC" symbols, i.e. native USB CDC is
  compiled in. On this board you talk to the REPL either over the CH343P
  USB-UART bridge (a plain COM port, always works) or over the S3's native USB.
- The board's SoC is the ESP32-S3R2: 2MB **quad** PSRAM (standard variant, not
  SPIRAM_OCT). All three candidate images (official, Waveshare demo, gc9a01
  prebuilt) contain `quad_psram` symbols, and the official page says PSRAM is
  auto-detected at startup. So the plain `ESP32_GENERIC_S3` variant is right;
  do NOT flash the SPIRAM_OCT variant.

### Alternative: Waveshare's demo image (custom build with frozen GC9A01 driver)

- Waveshare publishes a MicroPython image for this exact board, bundled in the
  official demo zip: `ESP32-S3-Touch-LCD-1.28-Demo/Firmware/MicroPython-bin/S3-Touch-LCD-1.28-MPY.bin`
  (inside https://files.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28/ESP32-S3-Touch-LCD-1.28-Demo.zip).
- Scanning that binary shows it is "MicroPython-1.22.0-preview-xtensa-IDFv5.0.2"
  built 2023-12-02, with the gc9a01 C module compiled in (strings include
  `gc9a01.c` and `gc9a01_GC9A01_draw`) plus TinyUSB CDC.
- The wiki says of it: "the firmware is merged into a single file, note that the
  download address is the 0x00 location" (section 5.1.2 Flash the firmware).
- Trade-off: it is an old preview build (Dec 2023) frozen at MicroPython 1.22
  with an outdated frozen gc9a01 driver. Prefer the official v1.29.0 build plus
  a separately managed driver (next section). Use the Waveshare image only if
  you want a zero-effort smoke test or need their exact demo behavior.

## 2. Flashing on Windows 11: exact esptool commands

Sources: https://micropython.org/download/ESP32_GENERIC_S3/ installation text,
https://docs.micropython.org/en/latest/esp32/tutorial/intro.html (troubleshooting
and manual download mode), https://docs.espressif.com/projects/esptool/en/latest/esptool/basic-commands.html
(write-flash syntax; current esptool uses `write-flash` with a hyphen, legacy
`write_flash` also still works).

Install esptool first (one of):

```bash
pip install esptool
# or, on Windows where the entry point may be esptool.exe:
py -3.11 -m pip install esptool
```

Find the COM port: Device Manager -> Ports (COM and LPT), look for the CH343P
USB-UART bridge (Waveshare FAQ suggests `devmgmt.msc`). Typical value: `COM5`.

**BOOT button ritual (only needed if auto-boot fails):** hold BOOT, press and
release RESET, release BOOT. The board then sits in the ROM bootloader download
mode and esptool can talk to it. The Waveshare FAQ (same wiki page, section 8)
phrases it: "Long press the BOOT button, press RESET at the same time, then
release RESET, then release the BOOT button, at this time the module can enter
the download mode". The MicroPython docs give the identical sequence for boards
with BOOT and RESET buttons.

**Erase (first flash on a board that shipped with factory firmware):**

```bash
esptool --port COM5 erase_flash
```

(The MicroPython download page says: "If you are putting MicroPython on your
board for the first time then you should first erase the entire flash using:
esptool.py erase_flash"; on Windows the program may be `esptool` rather than
`esptool.py`.)

**Flash the firmware at offset 0:**

```bash
esptool --port COM5 --baud 460800 write_flash 0 ESP32_GENERIC_S3-20260824-v1.29.0.bin
```

(MicroPython page: "deploy the firmware to the board, starting at address 0";
"if flashing starts and then fails partway through, try removing the --baud
460800 option".)

**Sanity checks and recovery:**

- `esptool --port COM5 flash_id` prints the flash chip; on this board expect
  16MB (0x1000000). If it prints 8MB, the board variant differs from the wiki
  and the 8MB gc9a01 prebuilt should be used instead (see below).
- Reset after flashing: press RESET once. If the port disappears or flashing
  fails repeatedly, do the BOOT ritual and retry; the Waveshare FAQ also notes
  "Click the Reset button for more than 1 second, wait for the PC to re-recognize
  the device and then download again".
- The Waveshare wiki points Windows users who prefer GUIs at the Espressif
  Flash Download Tool (https://files.waveshare.com/wiki/common/Flash_download_tool_3.9.5_0.zip)
  with download address 0x00, but esptool is the scriptable path.

## 3. Display driver: GC9A01 under MicroPython

### Primary choice: compiled `gc9a01` C module from russhughes/gc9a01_mpy

- Repo: https://github.com/russhughes/gc9a01_mpy ("Fast MicroPython driver for
  GC9A01 display modules written in C", 216 stars at research time). Based on
  devbis' st7789_mpy. Docs: https://russhughes.github.io/gc9a01_mpy/
- **Version**: the repo has NO git tags and NO GitHub releases (verified via
  the GitHub API: `releases` returns an empty array, `tags` returns empty). Pin
  by commit instead. Last push to the repo was 2025-03-24 (main branch). State
  this pin explicitly in the firmware repo when vendoring.
- **Install path (recommended)**: use the prebuilt firmware image from the
  repo's `firmware/ESP32_GENERIC_S3/` directory, which bakes the compiled
  `gc9a01` module (plus frozen fonts) into the image:
  - File: `firmware/ESP32_GENERIC_S3/firmware_16MiB.bin`
    (https://github.com/russhughes/gc9a01_mpy/tree/main/firmware/ESP32_GENERIC_S3)
  - Scanned binary reports: "MicroPython v1.22.2-dirty on 2024-02-25",
    IDF v5.0.4, with the gc9a01 C module compiled in and quad PSRAM support.
  - For this board pick the 16MiB file (the board has 16MB flash per section 0).
    4/8/16/32MiB variants all exist; all are the same size image, only the
    flash-size bootloader assumptions differ.
  - This image is flashed exactly like the official one (section 2), at offset 0.
- **Install path (if you stay on official v1.29.0 firmware)**: there is no
  `.mpy` distribution, so you must either (a) build the module yourself with
  mpremote/mcross or the ESP-IDF toolchain following the repo's build docs, or
  (b) fall back to the pure-Python driver below. Building the C module for the
  exact MicroPython version on the device is a known pain point (ABI must
  match), which is why the prebuilt image or the pure-Python driver are the
  two pragmatic options for lovebird v0.

### Fallback: pure-Python driver from russhughes/gc9a01py

- Repo: https://github.com/russhughes/gc9a01py ("GC9A01 Display driver in
  MicroPython", 62 stars; a fork of devbis' st7789py_mpy). Install = copy
  `gc9a01py.py` to the board's filesystem (e.g. `/lib/gc9a01py.py`) via
  Thonny, mpremote or rshell. Slower than the C module (pure Python frame
  pushing), but zero build steps and fine for a tamagotchi's 240x240 UI.
- Note: the repo is old (last push 2021) but the GC9A01 init sequences it
  carries are the standard ones; Waveshare's own Arduino demo uses the same
  command set (see TFT_eSPI `GC9A01_Init.h` in the demo zip).

### Known-good constructor (pins verified against the wiki pin table)

Waveshare's own MicroPython demo `alien.py` (from the demo zip,
`ESP32-S3-Touch-LCD-1.28-Demo/MicroPython/alien.py`) constructs the display
like this, and these pins match the wiki pin table exactly:

```python
from machine import Pin, SPI
import gc9a01

tft = gc9a01.GC9A01(
    SPI(2, baudrate=80000000, polarity=0, sck=Pin(10), mosi=Pin(11)),
    240,
    240,
    reset=Pin(14, Pin.OUT),
    cs=Pin(9, Pin.OUT),
    dc=Pin(8, Pin.OUT),
    backlight=Pin(2, Pin.OUT),
    rotation=0,
)
tft.init()
```

## 4. Touch driver: CST816S in MicroPython

- Chip: Hynitron CST816S capacitive touch, I2C address 0x15, on this board
  wired as TP_SDA=GPIO6, TP_SCL=GPIO7, TP_INT=GPIO5, TP_RST=GPIO13 (wiki pin
  table). Touch interface per wiki specs table: I2C.
- **A MicroPython driver already exists and ships in the Waveshare demo zip**:
  `ESP32-S3-Touch-LCD-1.28-Demo/MicroPython/cst816.py` (5536 bytes). It is
  derived from NeoStormer's CircuitPython_CST816 (MIT license, copyright 2023
  NeoStormer, repo https://github.com/NeoStormer/CircuitPython_CST816),
  adapted by Waveshare for this board with pins hardcoded: `I2C(0, scl=Pin(7),
  sda=Pin(6), freq=400000)` and reset on `Pin(13)`. It exposes `who_am_i()`
  (chip ID 0xB5 check), `get_point()` (x/y from registers 0x03-0x06),
  `get_gesture()` (register 0x01: up/down/left/right/click/double click/long
  press), `get_touch()` (finger count > 0), `get_distance()`, plus gesture
  constants. The companion `cst816s_example.py` prints point/gesture/press
  continuously.
- **Port plan**: vendor `cst816.py` into the lovebird firmware repo (MIT, so
  license-compatible), then harden it: move the hardcoded pins into
  constructor parameters, drive TP_INT (GPIO5) for interrupt-driven reads
  instead of polling, and add proper power management hooks (the driver has
  wake_up/stop_sleep). Waveshare's C demo (Arduino `CST816S.cpp` in the same
  zip) is the reference for register semantics if deeper behavior is needed.
- Other MicroPython-ish options found by search (all secondary): the
  tillig-th/micropython-esp32-s3-c6-st7789-cst816-axs5106l-Touch-Boards repo
  (0 stars, ST7789 boards, driver for CST816) exists but the Waveshare-bundled
  driver is better-proven for this exact board. No high-star dedicated
  "CST816S for MicroPython" library exists as of research time (GitHub search
  for cst816s sorted by stars returns mostly Arduino C++/Rust/Linux drivers;
  fbiego/CST816S is Arduino, lupyuen's are C for other RTOSes).

## 5. Reading the five-way joystick

**There is no five-way joystick on this board.** Verified: the wiki pin table
and specs contain no joystick entry; the product page lists only "RESET
button" and "BOOT button"; the pin table's spare GPIOs (15-18, 21, 33 on the
SH1.0 1.0mm header, plus GPIO1 battery ADC, GPIO3/4 IMU interrupts, GPIO4/5
MOSFET chip-select pads) are the only expansion. Input on this board is:

- capacitive touch (CST816S, coordinates + built-in gestures), and
- the QMI8658 IMU (I2C on GPIO6/7, INT1=GPIO4, INT2=GPIO3) for
  shake/tilt/flip gestures, which is how the firmware skeleton should
  implement "pet reacts to being picked up".

If physical buttons turn out to be wanted for v0, they must be added
externally on the SH1.0 header GPIOs (15-18, 21, 33) - they are not on board.

## 6. Pin table (from the Waveshare wiki, verified 2026-11-02)

Source: https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28, "Pin
connection" table. Columns are the wiki's own (LCD, SH1.0 header, MX1.25
battery, QMI8658 IMU, other).

| ESP32-S3 pin | LCD | SH1.0 | MX1.25 | QMI8658 | other |
|---|---|---|---|---|---|
| GPIO0  | | | | | BOOT0 |
| GPIO1  | | | ADC (battery voltage) | | |
| GPIO2  | LCD_BL (backlight PWM) | | | | |
| GPIO3  | | | | INT2 | |
| GPIO4  | | | | INT1 | MOSFET1_CS |
| GPIO5  | TP_INT | | | | MOSFET2_CS |
| GPIO6  | TP_SDA | | | SDA | |
| GPIO7  | TP_SCL | | | SCL | |
| GPIO8  | LCD_DC | | | | |
| GPIO9  | LCD_CS | | | | |
| GPIO10 | LCD_CLK | | | | |
| GPIO11 | LCD_MOSI | | | | |
| GPIO12 | LCD_MISO | | | | |
| GPIO13 | TP_RST | | | | |
| GPIO14 | LCD_RST | | | | |
| GPIO15 | | GPIO15 | | | |
| GPIO16 | | GPIO16 | | | |
| GPIO17 | | GPIO17 | | | |
| GPIO18 | | GPIO18 | | | |
| GPIO19 | | | | | |
| GPIO20 | | | | | |
| GPIO21 | | GPIO21 | | | |
| GPIO33 | | GPIO33 | | | |

Additional wiki facts: USB-C carries the CH343P USB-UART on GPIO43 (TXD) /
GPIO44 (RXD); battery voltage divides 200K/100K into GPIO1 (formula in wiki:
`3.3 / (1<<12) * 3 * AD_Value`); display interface SPI (4-wire, up to 80MHz
per wiki); touch I2C; IMU I2C on the same bus (0x6A typical, see QMI8658
datasheet in the wiki resources).

## 7. Known-good minimal main.py (draw + touch)

Verified pieces: the gc9a01 constructor pins are copied verbatim from
Waveshare's working `alien.py` demo, the cst816 usage from Waveshare's
`cst816s_example.py`, and both match the wiki pin table. Combined outline:

```python
# main.py for Waveshare ESP32-S3-Touch-LCD-1.28 under MicroPython
# display: compiled gc9a01 module (russhughes/gc9a01_mpy) or the
#          Waveshare demo image; touch: cst816.py from the Waveshare demo
import time
from machine import Pin, SPI
import gc9a01
import cst816

tft = gc9a01.GC9A01(
    SPI(2, baudrate=80000000, polarity=0, sck=Pin(10), mosi=Pin(11)),
    240, 240,
    reset=Pin(14, Pin.OUT), cs=Pin(9, Pin.OUT), dc=Pin(8, Pin.OUT),
    backlight=Pin(2, Pin.OUT), rotation=0)
tft.init()

touch = cst816.CST816()
if not touch.who_am_i():
    raise RuntimeError("CST816S not detected on I2C")

BLACK = gc9a01.BLACK
WHITE = gc9a01.WHITE
tft.fill(BLACK)
tft.text("lovebird", 60, 110, WHITE, BLACK)   # with a bitmap font from the driver

last_point = None
while True:
    if touch.get_touch():
        p = touch.get_point()
        if (p.x_point, p.y_point) != last_point:
            tft.fill_rect(p.x_point - 4, p.y_point - 4, 8, 8, gc9a01.RED)
            last_point = (p.x_point, p.y_point)
    g = touch.get_gesture()
    if g:
        # 1 up, 2 down, 3 left, 4 right, 5 click, 11 double click, 12 long press
        tft.text(str(g), 8, 8, WHITE, BLACK)
    time.sleep_ms(50)
```

Caveats: `tft.text` needs one of the driver's bitmap fonts (see gc9a01_mpy
docs / the demo zip's `bitmap/` directory); with the pure-Python `gc9a01py`
driver the constructor is identical but drawing is slower; interrupt-driven
touch via TP_INT (GPIO5) is the v1 improvement over polling.

## 8. Reference projects on round Waveshare boards (ticket asked for TamaPoke / TamaFi)

- **TamaPoke** (https://github.com/socquique/TamaPoke, 267 stars): a Pokemon
  tamagotchi, but for the **Waveshare ESP32-S3-Touch-AMOLED-1.75** (round 466x466
  AMOLED, CO5300 driver over QSPI, CST9217 touch). It is a different round
  board: its display and touch drivers do NOT transfer to the LCD-1.28. Useful
  as reference for game structure, MQTT-less single-player loop and the
  browser-based installer pattern, not for drivers.
- **TamaFi** (https://github.com/cifertech/TamaFi, 475 stars): an ESP32-S3
  virtual pet with an ST7789 240x240 TFT, Arduino/ESP-IDF (C++), not
  MicroPython. Useful as a reference for pet-state machine and menu design;
  display driver not transferable (ST7789 != GC9A01).
- The project the Waveshare wiki itself links for this exact board is
  **Luna - the Live Moon Phase Display** (ElectroDonut digital soul pendant)
  by vishalsoniindia: https://github.com/vishalsoniindia/Luna---The-Live-Moon-Phase-Display
  (Wi-Fi moon phase display on this round board; README-verified). Its repo
  shows a working bring-up of this display on this board and is the closest
  public reference implementation.
- Waveshare's own demo zip (section 1/3/4 above) remains the canonical
  working reference for this exact board: its MicroPython demos (alien.py,
  hello.py, cst816s_example.py, jpg.py, rotation.py, scroll.py, etc.) all run
  on the demo image, and its constructor snippets are what section 7 reuses.

## 9. Facts that could NOT be established from primary sources

Stated explicitly per the ticket:

- **Exact PSRAM topology of the ESP32-S3R2 (quad vs octal) from an Espressif
  datasheet page**: Espressif's datasheet PDFs are behind a JS-rendered docs
  site that could not be fetched cleanly. However, three independent primary
  artifacts (the Waveshare demo image, the russhughes 16MiB prebuilt, and the
  official MicroPython v1.29.0 binary) all contain `quad_psram` symbols and no
  octal-PSRAM-only code path, and the official MicroPython page says the
  firmware "will auto-detect a connected SPIRAM chip at startup". Conclusion:
  use the standard (non-OCT) build. Confidence high, but the datasheet itself
  was not directly quoted.
- **A tag/version number for russhughes/gc9a01_mpy**: none exists (no tags,
  no releases; verified via the GitHub API). Pin by commit hash when vendoring;
  the prebuilt images self-identify as "MicroPython v1.22.2-dirty on
  2024-02-25".
- **Whether any 8MB-flash variant of this board exists**: the wiki and product
  page only ever list the 16MB W25Q128JVSIQ. If a unit ever reports 8MB via
  `esptool flash_id`, the 8MiB firmware variants referenced in sections 1 and 3
  exist for that case; but no primary source documents an 8MB SKU.

## Source index

- Waveshare wiki (board): https://www.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28
- Waveshare product page: https://www.waveshare.com/esp32-s3-touch-lcd-1.28.htm
- Waveshare demo zip: https://files.waveshare.com/wiki/ESP32-S3-Touch-LCD-1.28/ESP32-S3-Touch-LCD-1.28-Demo.zip
- MicroPython ESP32_GENERIC_S3 downloads: https://micropython.org/download/ESP32_GENERIC_S3/
- MicroPython ESP32 port board list: https://micropython.org/download/?port=esp32
- MicroPython ESP32 tutorial intro: https://docs.micropython.org/en/latest/esp32/tutorial/intro.html
- esptool basic commands: https://docs.espressif.com/projects/esptool/en/latest/esptool/basic-commands.html
- russhughes/gc9a01_mpy: https://github.com/russhughes/gc9a01_mpy (docs https://russhughes.github.io/gc9a01_mpy/)
- russhughes/gc9a01py: https://github.com/russhughes/gc9a01py
- NeoStormer/CircuitPython_CST816: https://github.com/NeoStormer/CircuitPython_CST816
- TamaPoke: https://github.com/socquique/TamaPoke
- TamaFi: https://github.com/cifertech/TamaFi
- Luna moon phase pendant (this exact board): https://github.com/vishalsoniindia/Luna---The-Live-Moon-Phase-Display