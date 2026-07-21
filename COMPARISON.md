I’ll compare the two driver implementations at the hardware boundary: bus type, I2C address, reset/interrupt wiring, initialization tables, coordinate/report format, and any chip-specific assumptions. I’ll first read both source/header sets and their local READMEs, then summarize concrete differences and note any mismatches or likely naming quirks.

...
<!--
Read [](esphome/gsl3670/touchscreen.py)

Read [](esphome/gsl3670/gsl3670_touchscreen.cpp)

Read [](esphome/gsl3670/gsl3670_touchscreen.h)

Read [](esp-idf/esp_lcd_touch_gsl3680/esp_lcd_touch_gsl3680.c#1-1), lines 1 to 420

Read [](esp-idf/esp_lcd_touch_gsl3680/include/esp_lcd_touch_gsl3680.h#1-1), lines 1 to 260

Read [](esp-idf/esp_lcd_touch_gsl3680/gsl_point_id.c#1-1), lines 1 to 320

Read [](devices/guition-jc8012p4a1/README.md#1-1), lines 1 to 260

Read [](esphome/gsl3670/__init__.py)
-->

The shared hardware shape is already clear: both are I2C controllers at `0x40`, read touch data from register `0x80`, and use the same 4-byte-per-contact encoding. The meaningful differences are in firmware/configuration handling, reset/INT sequencing, post-read filtering, and framework integration; I’m now checking the remaining ESP-IDF initialization and I2C helper code to distinguish actual device behavior from API boilerplate.

...
<!--
Searched for regex `touch_gsl3680_(reset|read_cfg|i2c_read|i2c_write|load_fw|clear_reg|startup_chip|init|read_ram_fw)|esp_lcd_touch_gsl3680_(load_fw|clear_reg|startup_chip|init|read_ram_fw)` (`**/esp-idf/esp_lcd_touch_gsl3680/esp_lcd_touch_gsl3680.c`), 48 results

Searched for regex `GSLX680_FW|gsl_config_data_id|gsl_mask_tiaoping|gsl_alg_id_main|MAX_FINGER_NUM|dev_addr|rst_gpio_num|int_gpio_num` (`**/esp-idf/esp_lcd_touch_gsl3680/**`), 44 results

Read [](esp-idf/esp_lcd_touch_gsl3680/esp_lcd_touch_gsl3680.c#400-400), lines 400 to 701

Read [](esp-idf/esp_lcd_touch_gsl3680/CMakeLists.txt)

Read [](esp-idf/esp_lcd_touch_gsl3680/idf_component.yml)
-->

The ESP-IDF path confirms that its “GSL3680” implementation loads a compiled `GSLX680_FW` table plus a separate `gsl_config_data_id`, then runs a coordinate/ID correction algorithm on every report. The ESPHome path instead consumes an externally supplied 132-byte-record firmware blob and forwards decoded raw points; I’m doing one final pass over configuration metadata to separate board defaults from driver-level requirements.

...
<!--
Searched for regex `gsl3670|gsl3680|touch|i2c|interrupt|reset|0x40|GPIO` (`**/*.{md,py,cpp,h,c,yml,yaml}`), 525 results

Read [](esphome/gsl3670/__init__.py)

Read [](esp-idf/esp_lcd_touch_gsl3680/include/gsl_point_id.h)

Read [](README.md)
-->

Both drivers use the same basic electrical interface and touch-report protocol. The main differences are the firmware/configuration payload, GPIO sequencing, and coordinate post-processing.

**Short answer**

| Aspect | ESPHome `gsl3670` | ESP-IDF `gsl3680` |
|---|---|---|
| Bus | I2C | I2C through `esp_lcd_panel_io` |
| I2C address | `0x40` | `0x40` |
| Touch-data register | `0x80` | `0x80` |
| Touch report size | Reads 44 bytes | Reads 44 bytes |
| Contact encoding | Same GSL format: X/Y plus 4-bit ID | Same |
| Maximum contacts handled | 3, explicitly clamped | Intended maximum 3, but input count is not clamped before parsing |
| Reset sequence | GPIO reset plus GSL register sequence | Same logical sequence, with longer delays |
| Interrupt | Optional ESPHome interrupt, falling-edge | Configurable ESP-IDF interrupt polarity |
| Firmware | External, downloadable or local firmware blob | Firmware table compiled into the driver |
| Coordinate processing | Raw coordinates passed to ESPHome | Coordinates passed through `gsl_point_id` filtering/correction |
| Address strap handling | None | Temporarily drives the interrupt pin during startup to select/check the address |

**I2C address and bus**

The addresses are not different. ESPHome applies the address through its I2C device schema, with a default of `0x40` in touchscreen.py. The ESP-IDF driver defines the same address in esp_lcd_touch_gsl3680.h.

There is no serial, SPI, or network “port” difference here. Both drivers communicate with the touch controller over I2C.

The ESP-IDF driver uses the ESP LCD panel-I/O abstraction:

- Reads use `esp_lcd_panel_io_rx_param()`.
- Writes use `esp_lcd_panel_io_tx_param()`.

That is an API/framework difference, not a device-interface difference. See esp_lcd_touch_gsl3680.c.

**Touch data format**

The two implementations read the same register and the same amount of data:

- Register `0x80`
- 44 bytes
- First byte is the reported finger count
- Each contact occupies four bytes
- The upper nibble contains the contact ID
- The lower nibble and following byte contain X
- The first two bytes contain Y

ESPHome decodes this directly in gsl3670_touchscreen.cpp. The ESP-IDF driver performs essentially the same decoding in esp_lcd_touch_gsl3680.c.

So despite the `3670` versus `3680` names, the two drivers assume a very similar GSL controller register protocol.

**Firmware and initialization data**

This is the largest device-level difference.

The ESPHome driver does not contain a fixed firmware table. Its YAML/code-generation layer downloads or reads a firmware blob and embeds it into the generated firmware. The default Seeed model points at `seeed-d1001-fw.bin` in touchscreen.py.

The blob is expected to consist of records of:

```text
4-byte page/address header
128-byte payload
```

The ESPHome driver writes each record by:

1. Writing the first byte to register `0xf0`.
2. Writing the following 128 bytes starting at register `0x00`.

That behavior is in gsl3670_touchscreen.cpp.

The ESP-IDF driver instead contains a statically compiled `GSLX680_FW` table in esp_lcd_touch_gsl3680.h. Each table entry consists of:

```text
register offset
32-bit value
```

The ESP-IDF loader writes one 32-bit value at a time, except for page-select entries at `0xf0`, which are written as one byte. See esp_lcd_touch_gsl3680.c.

Therefore:

- The programming mechanism is broadly the same GSL firmware-loading style.
- The actual firmware/configuration payloads are different.
- The ESPHome firmware is specifically associated with the Seeed D1001 model by default.
- The ESP-IDF firmware table is labeled `GSLX680_FW`, even though the surrounding driver is called `gsl3680`.

Those firmware payloads should not be assumed interchangeable merely because the runtime touch-data format is similar.

The ESP-IDF driver also initializes a separate `gsl_config_data_id` array through `gsl_DataInit()` in esp_lcd_touch_gsl3680.c. That configuration is used by the host-side coordinate/ID algorithm, not simply transmitted as the same firmware blob.

**Reset and startup sequence**

Both drivers implement the same high-level sequence:

```text
clear registers
reset controller
load firmware
start chip
reset controller again
start chip again
```

ESPHome performs it in gsl3670_touchscreen.cpp.

ESP-IDF performs it in esp_lcd_touch_gsl3680.c.

Both clear or initialize:

```text
0x88 <- 0x01
0xe4 <- 0x04
0xe0 <- 0x00
```

Both reset using:

```text
reset GPIO pulse
0xe4 <- 0x04
0xbc <- four zero bytes
```

The timing differs:

- ESPHome uses approximately 1 ms reset-low and 5 ms reset-high delays, with no delays between the register writes in the clear sequence.
- ESP-IDF uses approximately 20 ms reset-low and 20 ms reset-high delays, plus additional 5 to 20 ms delays between register writes.

Compare gsl3670_touchscreen.cpp with esp_lcd_touch_gsl3680.c and esp_lcd_touch_gsl3680.c.

This may matter on real hardware: the ESP-IDF version is more conservative about controller timing.

**Reset and interrupt GPIOs**

ESPHome allows both pins to be configured. For the built-in `SEEED-RETERMINAL-D1001` model, the defaults are:

- Reset: XL9535 expander pin 14
- Interrupt: GPIO 16

These defaults are defined in touchscreen.py. ESPHome configures the interrupt for a falling edge in gsl3670_touchscreen.cpp.

The ESP-IDF driver receives reset and interrupt GPIOs in `esp_lcd_touch_config_t`. During initialization it temporarily configures the interrupt pin as an output, drives it low, toggles reset, and then changes it back to an input with the configured interrupt polarity. This is in esp_lcd_touch_gsl3680.c.

That temporary interrupt-pin manipulation is a device-specific behavior absent from the ESPHome implementation. It appears intended to select or validate the GSL I2C address. However, the current ESP-IDF code only accepts `0x40`; its “alternate address” branch also sets the pin low and reports an error.

**Coordinate and touch processing**

ESPHome forwards decoded raw points almost directly:

- It limits the finger count to three.
- It rejects points where X or Y exceeds `8192`.
- It calls ESPHome’s `add_raw_touch_position_()`.

See gsl3670_touchscreen.cpp.

Axis transformations are configured at the ESPHome level. The Seeed model defaults include:

- Swap X/Y
- Mirror X
- Mirror Y
- X range: 20 to 872
- Y range: 20 to 1644

These are defined in touchscreen.py.

The ESP-IDF driver does more processing after reading the controller:

1. Decodes the raw contacts.
2. Calls `gsl_alg_id_main(&cinfo)`.
3. Calls `gsl_mask_tiaoping()`.
4. Potentially writes correction data back to registers `0xf0` and `0x08`.
5. Returns the corrected coordinates.

That processing occurs in esp_lcd_touch_gsl3680.c, with the algorithm implemented in gsl_point_id.c.

So the ESP-IDF driver includes filtering, point tracking, ID handling, interpolation, edge handling, and coordinate correction that the ESPHome driver does not duplicate internally.

**Potential implementation issues in the ESP-IDF driver**

A few details are worth noting:

- `MAX_FINGER_NUM` is 3, but `Finger_num` is copied from the hardware without being clamped before the parsing loop. A controller reporting more than three contacts could write past `XY_Coordinate`, which is declared with only three entries. See esp_lcd_touch_gsl3680.c and esp_lcd_touch_gsl3680.c.
- `touch_gsl3680_read_cfg()` initializes `i2c_buffer_read` to zero but never assigns the value read from the controller to it. Consequently, its comparison against `0x12` will always fail, although the caller currently ignores that function’s return value. See esp_lcd_touch_gsl3680.c.
- The ESP-IDF implementation uses a global `esp_lcd_touch_gsl3680` handle rather than keeping all state private to an instance. That is mainly an integration limitation, but it means multiple controllers would not be supported cleanly.

**Conclusion**

The drivers are not using different buses or different I2C addresses. They appear to target closely related GSL touch-controller protocols, and their low-level touch register access is nearly identical.

The important hardware behavior differences are:

1. They load different firmware/configuration payloads.
2. ESP-IDF uses longer reset and initialization delays.
3. ESP-IDF manipulates the interrupt pin during address selection/startup.
4. ESP-IDF applies a substantial touch-coordinate and ID correction algorithm.
5. ESPHome applies model-specific axis transforms and calibration through the ESPHome touchscreen framework.
6. ESPHome explicitly clamps the contact count to three; ESP-IDF currently does not.

The `gsl3670` and `gsl3680` names alone therefore do not prove that the chips are electrically or protocol-wise different. In this repository, the strongest practical distinction is that the ESPHome driver is paired with a Seeed D1001 firmware blob, while the ESP-IDF driver is paired with a compiled GSLX680-style firmware table and correction algorithm intended for the Guition panel.