# Guition JC8012P4A1

The Guition JC8012P4A1 is a 10.1" LCD display module with a display resolution of 800 x 1280 and a touchscreen, powered by an ESP32-P4 and a secondary ESP32-C6.

## Features

* 10.1-inch color screen that supports 24 bit RGB with 16.7M colors 
* 800x1280 screen resolution
* The SKU JC8012P4A1C_I_W_Y comes without battery
* The SKU JC8012P4A1C_I_W_Y1 comes with battery included
* TF card slot
* Lithium battery interface circuit 

## 10.1" ESP32-P4 HMI Display Module

> Preliminary datasheet compiled from manufacturer documentation,
> reverse engineering, ESPHome sources and community findings.

---

# General

| Property | Value |
|----------|------|
| Product | JC8012P4A1C-I-W-Y |
| Manufacturer | Jingcai / GUITION / DIYmalls |
| Module Type | ESP32-P4 HMI Display |
| MCU | ESP32-P4 |
| Wireless MCU | ESP32-C6 |
| Operating Voltage | 5 V |
| Typical Current | ≈700 mA |
| Weight | ≈550 g |

---

# Display

| Property | Value |
|----------|------|
| Size | 10.1 inch |
| Technology | TFT LCD |
| Panel Type | IPS |
| Color Depth | 24-bit (16.7M colors) |
| Resolution | 800 × 1280 |
| Orientation | Portrait |
| Active Area | 216.58 × 135.36 mm |
| Module Size | 242.8 × 158.7 mm |
| Aspect Ratio | 10:16 |
| Pixel Density | ≈149 PPI |
| Driver IC | JD9365 |
| Backlight | LED |

---

# LCD Controller

| Property | Value |
|----------|------|
| Controller | JD9365 |
| Interface | RGB Parallel |
| Color Format | RGB888 |
| Data Width | 24 Bit |
| Display RAM | Internal |
| Initialization | Vendor-specific register sequence |

---

# Touch Panel

| Property | Value |
|----------|------|
| Type | Capacitive Multi-Touch |
| Controller | CST226 (board dependent) |
| Interface | I²C |
| Touch Points | Multi-touch |

---

# ESP32-P4

| Property | Value |
|----------|------|
| CPU | Dual-core RISC-V |
| Frequency | up to 400 MHz |
| PSRAM | 32 MB |
| Flash | 16 MB |
| Wi-Fi | via ESP32-C6 |
| Bluetooth | BLE 5 |
| USB | USB-C |

---

# Interfaces

- USB-C
- UART
- Speaker
- Microphone
- Camera (MIPI CSI)
- TF Card
- LCD FPC
- Touch FPC
- Two GPIO Expansion Slots
- RTC Battery Connector

## GPIO Expansion Slot 1

- 3V3
- 3V3
- I2CSDA
- I2CSCL
- GPIO2
- GPIO3
- GPIO4
- GPIO5
- GPIO48
- GPIO47
- GPIO46
- GPIO45
- GND
- GND

## GPIO Expansion Slot 2

- 3V3
- 3V3
- GPIO34
- GPIO33
- GPIO32
- GPIO31
- GPIO30
- GPIO29
- GPIO28
- NC
- NC
- NC
- GND
- GND

---

# Supported Software

- ESP-IDF
- Arduino
- ESPHome
- LVGL
- MicroPython

---

# ESPHome Display Driver

Current implementation uses:

Driver:
```text
jd9365
```

Resolution:
```text
800 × 1280
```

Pixel Clock:
```text
70 MHz
```

RGB Timing

| Parameter | Value |
|-----------|------:|
| PCLK | 70 MHz |
| HSync Front Porch | 40 |
| HSync Pulse | 48 |
| HSync Back Porch | 40 |
| VSync Front Porch | 1 |
| VSync Pulse | 31 |
| VSync Back Porch | 13 |

Initialization

Uses the complete JD9365 initialization sequence included in ESPHome.

---

# Mechanical

| Property | Value |
|----------|------|
| Width | 242.8 mm |
| Height | 158.7 mm |
| Active Display | 216.58 × 135.36 mm |

---

# Environmental

| Property | Value |
|----------|------|
| Operating Temperature | -20°C … +70°C |
| Storage Temperature | -30°C … +80°C |

---

# Known Variants

| Model | Description |
|-------|-------------|
| JC8012P4A1C-I-W-Y | Capacitive Touch |
| JC8012P4A1C-I-W-Y1 | Capacitive Touch + Battery |

---

# Notes

The module exists with at least two different LCD panel revisions.

Known display controller:
- JD9365

Some LCD panels require different initialization sequences although the controller remains identical.

ESPHome includes a dedicated initialization sequence for the V2 hardware.

---

# Unknown / To Be Verified

- Exact LCD panel manufacturer
- LCD glass model number
- Backlight current
- Typical brightness (likely ~320 nit)
- Contrast ratio
- Gamma curve
- Color gamut
- Polarizer type
- Viewing angle specification
- Display response time
- Touch controller revision
- Display power sequencing