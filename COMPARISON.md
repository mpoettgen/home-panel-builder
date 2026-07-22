> The esphome and the esp-idf folders contain two different drivers. In the esphome folder you will find the gsl3670 driver for ESPHome. In the esp-idf folder you will find an esp-idf driver for the gsl3680. I know, that the drivers use different frameworks to do what they need to do. What I want to know is what the drivers do differently with regard to the device they are controlling. E.g. do they use different ports, addresses, initialization data, etc.

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

> I've written the embedded firmware from the esp-idf into a file using the same format as the seeed firmware and they are exactly the same.

That changes the conclusion significantly. If the ESP-IDF `GSLX680_FW` table, after conversion to the Seeed 132-byte-record format, is byte-for-byte identical, then the two drivers use the same embedded controller firmware. The apparent differences in firmware representation are only storage/loading format differences:

- ESP-IDF stores individual `offset + 32-bit value` records.
- ESPHome stores 128-byte page blocks prefixed by the page number.
- Both ultimately write the same GSL firmware contents to the controller.

The `gsl_config_data_id` array is separate. It is consumed by the ESP-IDF-side `gsl_point_id` algorithm through `gsl_DataInit()` and is not necessarily controller firmware. It affects host-side coordinate filtering, ID tracking, interpolation, and correction.

The revised hardware-level comparison is therefore:

- Same I2C bus.
- Same I2C address, `0x40`.
- Same touch-data register, `0x80`.
- Same 44-byte report layout.
- Same embedded controller firmware.
- Same general clear/reset/load/start sequence.
- Different reset timing.
- Different interrupt/address-selection startup handling.
- Different host-side touch processing.
- Different framework-level axis transforms and calibration.

This also makes it more likely that `gsl3670` and `gsl3680` are naming differences for closely related, or possibly identical, controller firmware/platform variants rather than evidence of different device initialization data. The important remaining question is whether the two drivers send the same bytes in the same order with compatible delays; the firmware contents alone do not establish that.

> The gsl3670 is supposedly for smaller displays (less than 10.1") with 26 TX, 14 RX and 364 sensor channels, while the gsl3680 is for slightly larger displays (less than 13.5") with 31 TX, 20 RX and 620 sensor channels. Is this in any way reflected by the different driver implementations? Can you please also check the folder esp32_p4_function_ev_board. It also contains code from an esp-idf demo project and may have interesting initialization data for the Guition JC8012P4A1 display for which it is used. Some of it will probably be different from the initialization data for the Seeed ReTerminal D1001.

I’ll verify this against the actual panel-demo sources rather than infer from the chip names. I’ll locate esp32_p4_function_ev_board, inspect its touch/display initialization and firmware/config data, then compare the hardware-specific values with the Seeed defaults and the two existing drivers.

...
<!--
Searched for files matching `home-panel-builder` · `**/esp32_p4_function_ev_board/**`, 13 matches

Searched for regex `TX|RX|sensor|GSL|gsl3680|gsl3670|touch|0x40|0x80|GSLX|fw|firmware|i2c`, 512 results

Read []()
-->

The demo folder is present, and its board README is generic, so the useful evidence will be in the board headers and C source. The search already shows that this component includes `esp_lcd_touch_gsl3680`; I’m narrowing in on the exact GPIO/I2C definitions, panel dimensions, touch constructor, and whether it carries a second firmware/configuration table.

...
<!--
Read [](/esp32_p4_function_ev_board/esp32_p4_function_ev_board.c#1-1), lines 1 to 260

Read [](/esp32_p4_function_ev_board/include/bsp/config.h)

Read [](/esp32_p4_function_ev_board/include/bsp/touch.h)

Read [](/esp32_p4_function_ev_board/include/bsp/display.h)

Read [](/esp32_p4_function_ev_board/include/bsp/esp32_p4_function_ev_board.h#1-1), lines 1 to 220

Read [](/esp32_p4_function_ev_board/README.md)

Searched for regex `bsp_touch_new|esp_lcd_touch_new_i2c_gsl3680|esp_lcd_new_panel_jd9365|jd9365|touch_config|panel_io|BSP_LCD_TOUCH` (`**/esp32_p4_function_ev_board/esp32_p4_function_ev_board.c`), 18 results

Searched for regex `jd9365|JD9365|init_cmds|reset|0x|mipi|1280|800` (`**/esp32_p4_function_ev_board/**`), 65 results

Read [](/esp32_p4_function_ev_board/esp32_p4_function_ev_board.c#260-260), lines 260 to 736

Read [](/esp32_p4_function_ev_board/idf_component.yml)
-->

The Guition board demo confirms the expected GSL3680 path is actually used for the 800x1280 panel: I2C SDA/SCL are GPIO 7/8, touch reset/INT are GPIO 22/21, address remains `0x40`, and the ESP-IDF touch configuration uses `x_max=800`, `y_max=1280`, mirror-Y only. It also points to the same local `esp_lcd_touch_gsl3680` component, so I’m checking whether the TX/RX/channel counts appear explicitly in its firmware/config tables or only indirectly through algorithm parameters.

...
<!--
Searched for regex `drv_num|sen_num|screen_x_max|screen_y_max|config_static\[|gsl_DataInit|CONFIG_LENGTH|sensor|channel|tx|rx` (`**/esp-idf/esp_lcd_touch_gsl3680/gsl_point_id.c`), 161 results

Searched for regex `26|14|31|20|364|620|0x1a|0x14|0x1f|0x0e` (`**/esp-idf/esp_lcd_touch_gsl3680/**`), 880 results

Searched for regex `26|14|31|20|364|620|tx|rx|sensor|channel` (`**/esphome/**`), 4 results

Read [](/esp-idf/esp_lcd_touch_gsl3680/gsl_point_id.c#2680-2680), lines 2680 to 3202
-->

The key finding is a direct contradiction to the proposed size split: the Guition demo’s `gsl_config_data_id` decodes to `drv_num=26`, `sen_num=14`, with `drv_num_nokey=26` and `sen_num_nokey=14`. In other words, the code labeled and used as GSL3680 for the Guition board is configured for 26 drive and 14 sense lines, not 31/20; I’m checking the missing local JD9365 component reference to see how much display initialization data is actually available here.

...
<!--
Searched for files matching `home-panel-builder` · `**/*jd9365*`, no matches

Searched for files matching `home-panel-builder` · `**/*{bin,trx,hex,dat}`, no matches

Read [](/esp32_p4_function_ev_board/Kconfig)

Read [](/esp-idf/esp_lcd_touch_gsl3680/include/esp_lcd_touch_gsl3680.h#20-20), lines 20 to 85
-->

Yes. The available code does reflect some panel-specific characteristics, but it does **not** support the proposed rule that GSL3670 means `26 TX / 14 RX` and GSL3680 means `31 TX / 20 RX`.

**Important finding**

The code used by the Guition ESP32-P4 demo is explicitly the `gsl3680` driver, but its `gsl_config_data_id` contains:

```text
drv_num       = 26
sen_num       = 14
drv_num_nokey = 26
sen_num_nokey = 14
```

The relevant packed values are:

```c
conf[0x12] = 0x000E001A
conf[0x13] = 0x000E001A
```

The ESP-IDF algorithm decodes them as:

```c
drv_num = conf[0x12] & 0xffff;        // 26
sen_num = conf[0x12] >> 16;           // 14
drv_num_nokey = conf[0x13] & 0xffff;  // 26
sen_num_nokey = conf[0x13] >> 16;     // 14
```

This decoding is implemented in gsl_point_id.c, while the Guition values are at the beginning of esp_lcd_touch_gsl3680.c.

So, in this repository:

- The driver called `gsl3680` is configured for `26 x 14`.
- I do not see `31`, `20`, or `620` used as the sensor matrix dimensions.
- The `620` figure is mathematically `31 x 20`, but it is not reflected in the Guition driver’s host-side configuration.
- The `364` figure is `26 x 14`, and that one is reflected directly.

This suggests that TX/RX count is a property of the particular touch-panel/controller firmware configuration, not a reliable universal distinction between the GSL3670 and GSL3680 part names.

**What the Guition demo actually configures**

The Guition board component uses the local GSL3680 driver:

```yaml
esp_lcd_touch_gsl3680:
  path: ../esp_lcd_touch_gsl3680
```

See idf_component.yml.

Its hardware configuration is:

| Setting | Guition ESP32-P4 demo |
|---|---:|
| I2C controller | Configurable ESP32-P4 I2C peripheral, default `1` |
| SDA | GPIO 7 |
| SCL | GPIO 8 |
| I2C speed | 400 kHz by default |
| Touch reset | GPIO 22 |
| Touch interrupt | GPIO 21 |
| I2C address | `0x40` |
| Display resolution | 800 x 1280 |
| Touch `x_max` | 800 |
| Touch `y_max` | 1280 |
| Touch flags | mirror Y |
| Swap XY | Disabled |

The pins are defined in esp32_p4_function_ev_board.h, and the I2C setup is in esp32_p4_function_ev_board.c.

The touch configuration is in esp32_p4_function_ev_board.c. In particular:

```c
.x_max = BSP_LCD_H_RES,       // 800
.y_max = BSP_LCD_V_RES,       // 1280
.rst_gpio_num = GPIO_NUM_22,
.int_gpio_num = GPIO_NUM_21,
.swap_xy = 0,
.mirror_x = 0,
.mirror_y = 1,
```

The low-level GSL algorithm itself uses:

```c
screen_x_max = 1280;
screen_y_max = 800;
```

from the packed value `0x03200500`. This looks reversed relative to the public touch configuration because of the panel’s coordinate orientation and the subsequent display/LVGL rotation handling. It is not evidence of a different I2C protocol.

**Comparison with the ESPHome Seeed configuration**

The ESPHome model has a substantially different calibration/axis configuration:

```python
CONF_SWAP_XY: True,
CONF_MIRROR_X: True,
CONF_MIRROR_Y: True,
CONF_X_MIN: 20,
CONF_Y_MIN: 20,
CONF_X_MAX: 872,
CONF_Y_MAX: 1644,
```

It also defaults to different GPIOs:

```python
CONF_RESET_PIN: {"xl9535": None, "number": 14},
CONF_INTERRUPT_PIN: 16,
```

These values are in touchscreen.py.

That means the two model configurations differ in:

- Physical reset pin
- Physical interrupt pin
- Axis swap
- X/Y mirroring
- Raw coordinate ranges
- Framework-level coordinate scaling

Those are clearly panel-specific settings. The Seeed values do not describe the Guition panel.

The Guition panel’s ESP-IDF setup uses GPIO 22/21 and an 800x1280 touch geometry. The Seeed ESPHome profile uses an I/O expander reset pin, GPIO 16 interrupt, and raw limits of approximately 872x1644.

**Initialization data**

There are three different kinds of initialization data to distinguish:

1. **Touch-controller firmware**
2. **Host-side GSL coordinate algorithm configuration**
3. **LCD display-controller initialization**

The first two are available in this workspace.

The GSL touch firmware in esp_lcd_touch_gsl3680.h is loaded by esp_lcd_touch_gsl3680.c. Based on your byte-for-byte conversion result, this is the same firmware data as the ESPHome Seeed blob, just represented differently.

The separate `gsl_config_data_id` is not controller firmware. It configures the host-side filtering and coordinate algorithm. It includes the `26/14` matrix configuration, screen dimensions, edge handling, ID tracking, filtering, and correction values. The algorithm consumes it via `gsl_DataInit()` in esp_lcd_touch_gsl3680.c.

The third category, the LCD initialization sequence, is separate from the touch driver. The Guition board code uses a JD9365 panel component:

```c
#include "esp_lcd_jd9365.h"
...
esp_lcd_new_panel_jd9365(...)
```

See esp32_p4_function_ev_board.c and esp32_p4_function_ev_board.c.

The board config identifies the panel as 800x1280 with two MIPI-DSI lanes at 1 Gbps in display.h. The actual `esp_lcd_jd9365` component is referenced as a sibling directory but is not present in this workspace, so its complete JD9365 register table cannot be compared here with the Seeed display initialization sequence.

There is also a naming inconsistency in the demo:

- The source includes and constructs `jd9365`.
- Some comments and the README call the 1280x800 option “ILI9881C”.

The code path itself is the stronger evidence: it invokes the JD9365 driver.

**Conclusion**

The TX/RX claim is only partially reflected:

- `26 TX / 14 RX` is directly present in the Guition `gsl3680` host-side configuration.
- `31 TX / 20 RX` is not present in the Guition code.
- The firmware itself is identical between the two drivers after format conversion.
- Therefore the GSL3670/GSL3680 names do not, by themselves, establish different sensor-matrix sizes.
- The major actual device differences in this repository are panel-specific calibration, GPIO wiring, display geometry, and LCD initialization.
- The Guition demo’s LCD initialization is separate from its touch initialization and is likely different from the Seeed ReTerminal’s display sequence, but the actual JD9365 component source is missing here, so that portion cannot be compared byte-for-byte.

> I've put another implementation of a gsl3680 driver for esphome into the esphome folder, next to the gsl370 driver. The thing I would like to figure out: would the gsl3670 driver also work for the gsl3680 and if so, how would it have to be configured. If not, what would have to be changed.

I’ll compare the newly added ESPHome GSL3680 implementation against the existing GSL3670 driver at the device-facing level, then trace what ESPHome configuration controls address, GPIOs, axes, matrix dimensions, and firmware. The key test is whether the GSL3670 code is protocol-compatible with the GSL3680 hardware or merely happens to share the same firmware/report format.

...
<!--
Searched for files matching `home-panel-builder` · `esphome/**`, 13 matches

Searched for regex `gsl3680|GSL3680|gsl3670|GSL3670|firmware|0x40|0x80|drv_num|sen_num|reset|interrupt` (`**/esphome/**`), 922 results

Read [](file:///home/michael/projects/home-panel-builder/esphome)
-->

The new GSL3680 ESPHome component is structurally much closer to the ESP-IDF driver: it bundles the firmware as `offset + 32-bit value` records and includes the point-ID algorithm. I’m reading its setup/report path and configuration schema now; that will tell us whether it is a true alternative implementation or whether the existing GSL3670 driver could be made equivalent with only YAML changes.

...
<!--
Searched for regex `gsl_DataInit|gsl_config_data_id|screen_x_max|screen_y_max|drv_num =|sen_num =|0x000e001a|0x03200500|0x200` (`**/esphome/gsl3680/**`), 63 results

Read [](file:///home/michael/projects/home-panel-builder/esphome/gsl3680/gsl3680.cpp)

Read [](file:///home/michael/projects/home-panel-builder/esphome/gsl3680/gsl3680.h)

Read [](file:///home/michael/projects/home-panel-builder/esphome/gsl3680/touchscreen.py)

Read [](file:///home/michael/projects/home-panel-builder/esphome/gsl3680/README.md)
-->

Yes, the existing ESPHome `gsl3670` driver would probably communicate with the Guition GSL3680 successfully, but only under specific conditions. The two drivers are compatible at the low-level controller protocol, not equivalent in behavior.

The important evidence is:

- Both use I2C address `0x40`.
- Both read touch data from register `0x80`.
- Both use the same GSL firmware.
- Both decode the same four-byte contact format.
- The new ESPHome GSL3680 component uses the same firmware table and the same `gsl_point_id` algorithm as the ESP-IDF GSL3680 driver.
- The existing GSL3670 driver already implements the same basic firmware boot sequence and raw touch decoding.

So the GSL3670 driver is not inherently limited to a GSL3670 silicon device. In this repository, the name mostly identifies the ESPHome component and its default firmware/configuration profile.

**What would need to be configured**

For the Guition JC8012P4A1, the existing GSL3670 component would need a model profile equivalent to the following:

```yaml
touchscreen:
  - platform: gsl3670
    model: CUSTOM
    i2c_id: bus_a
    reset_pin:
      number: GPIO22
    interrupt_pin:
      number: GPIO21
    swap_x_y: false
    mirror_x: false
    mirror_y: true
    x_min: 0
    y_min: 0
    x_max: 800
    y_max: 1280
```

The exact surrounding ESPHome syntax depends on the project’s display and I2C configuration, but these are the relevant device values from the Guition BSP:

```c
.x_max = 800,
.y_max = 1280,
.rst_gpio_num = GPIO_NUM_22,
.int_gpio_num = GPIO_NUM_21,
.swap_xy = 0,
.mirror_x = 0,
.mirror_y = 1,
```

They are defined in esp32_p4_function_ev_board.c, while the physical I2C pins are:

```text
SDA = GPIO7
SCL = GPIO8
I2C address = 0x40
```

from esp32_p4_function_ev_board.h and esp_lcd_touch_gsl3680.h.

The existing GSL3670 component can already select `CUSTOM`, which avoids the Seeed-specific defaults:

```python
" CUSTOM": {}
```

More precisely, the current key is `"CUSTOM"` in touchscreen.py. With `CUSTOM`, the reset pin, interrupt pin, axis transforms, coordinate limits, and firmware must be supplied explicitly.

The firmware must be the byte-for-byte-equivalent blob you generated from the ESP-IDF table. It should be supplied as:

```yaml
firmware:
  file: /path/to/guition-gsl3680-firmware.bin
```

The existing GSL3670 component validates that the file uses 132-byte blocks:

```text
4-byte page/address header
128-byte data block
```

That validation is in touchscreen.py.

**What the existing GSL3670 driver does not use**

The major functional difference is the missing host-side point-processing algorithm.

The new GSL3680 component does this after reading raw contacts:

```cpp
gsl_alg_id_main(&cinfo);
unsigned int mask = gsl_mask_tiaoping();
```

See gsl3680.cpp.

The algorithm is implemented in gsl_point_id.cpp, and it is configured by the `gsl_config_data_id` table in gsl3680_firmware.h.

The existing GSL3670 driver does not run that algorithm. It simply:

1. Reads 44 bytes from `0x80`.
2. Decodes up to three contacts.
3. Rejects coordinates above `8192`.
4. Passes raw points to ESPHome.

See gsl3670_touchscreen.cpp.

That means the GSL3670 driver may work for basic single-touch operation, but it will not behave identically to the GSL3680 implementation for:

- Multi-touch tracking
- Contact-ID stability
- Filtering
- Prediction
- Edge correction
- Point interpolation
- Noise suppression
- Dynamic mask correction

The `26 TX / 14 RX` values are consumed by this algorithm, not by the raw I2C protocol. In the new component’s point-ID configuration, the values are decoded from the packed configuration table and used throughout gsl_point_id.cpp. The existing GSL3670 component has no equivalent configuration path.

**Initialization sequence differences**

The existing GSL3670 driver performs:

```text
clear registers
reset
load firmware
start chip
reset again
start chip again
```

See gsl3670_touchscreen.cpp.

The new GSL3680 component performs:

```text
read configuration
clear registers
reset
load firmware
start chip
read RAM marker
initialize host-side point algorithm
```

See gsl3680.cpp.

The new driver also checks:

```text
register 0xf0
RAM register 0xb0
```

and expects `0x5a` marker bytes from RAM. The existing GSL3670 driver has no equivalent readback validation.

The existing GSL3670 initialization is therefore less diagnostic, but its extra reset/start cycle is not necessarily wrong. It mirrors the original Seeed-style initialization sequence. Since you confirmed that the firmware is identical, the controller should accept the sequence, but the GSL3680 version is preferable for the Guition board because it was built around this exact driver/configuration path and verifies the result.

**Touch-count behavior**

The existing GSL3670 driver supports up to three points:

```cpp
static const size_t MAX_TOUCHES = 3;
```

and reads all 44 bytes.

The new GSL3680 ESPHome implementation deliberately reads only 24 bytes and clamps to two contacts:

```cpp
if (touch_count > 2) {
  touch_count = 2;
}
```

See gsl3680.cpp.

That appears to be an implementation choice for the intended screensaver/wake use case, not a GSL3680 hardware limit. The ESP-IDF version reads 44 bytes and defines three coordinate slots, although it has its own bounds issue if the controller reports more than three.

Therefore, using the GSL3670 raw reader could actually expose more contacts than the new ESPHome GSL3680 component, but without the GSL point-ID algorithm those contacts may be less stable.

**Recommended choice**

For the Guition panel, use the new `gsl3680` component when possible. It has the correct:

- Firmware representation
- Guition-oriented boot/readback sequence
- Point-ID algorithm
- `gsl_config_data_id`
- GSL3680-specific implementation structure

Use the existing `gsl3670` component only if you specifically want a simpler raw-touch driver. To make that work reliably on the Guition panel:

1. Use the converted firmware file that you verified identical to the ESP-IDF firmware.
2. Select `model: CUSTOM`.
3. Set reset to GPIO22.
4. Set interrupt to GPIO21.
5. Use I2C address `0x40`.
6. Configure the Guition axis orientation: no swap, no X mirror, Y mirror.
7. Set coordinate limits appropriate for the 800x1280 panel.
8. Expect raw, unfiltered touch data rather than the GSL3680 point-ID behavior.
9. Prefer the longer reset/write delays used by the GSL3680 component if startup reliability is inconsistent.

**If the GSL3670 driver needs to be made equivalent**

The smallest useful code change would be to add the GSL3680 point-ID algorithm to the existing GSL3670 driver:

```cpp
#include "gsl_point_id.h"
#include "gsl3680_firmware.h"
```

Then, after decoding the raw contacts:

```cpp
gsl_alg_id_main(&cinfo);
unsigned int mask = gsl_mask_tiaoping();
```

and copy the mask writeback behavior from gsl3680.cpp.

It would also need to call:

```cpp
gsl_DataInit(gsl_config_data_id);
```

during setup, as the GSL3680 implementation does in gsl3680.cpp.

At that point, however, the GSL3670 and GSL3680 components would be converging into one driver. The cleaner design would be to retain one generic GSL driver with model-specific profiles for:

- Firmware
- Reset/interrupt pins
- Raw coordinate orientation
- Screen dimensions
- Point-ID configuration
- Maximum reported contacts
- Boot verification and timing

So the practical answer is: **yes, the GSL3670 driver should be able to operate the GSL3680/Guition hardware with the correct firmware and GPIO/axis configuration, but it will not provide the same touch quality or behavior until the GSL3680 point-ID algorithm and configuration table are added.**