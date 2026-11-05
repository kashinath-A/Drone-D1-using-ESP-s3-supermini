# ESP32-S3 Quadcopter Flight Controller

> A DIY brushed micro-quad built on an **ESP32-S3**, an **MPU6050** IMU, and **MOSFET motor drivers**, running the open-source **ESP-FC** firmware.
>
> **Designed and built by Kashi**

![status](https://img.shields.io/badge/status-work%20in%20progress-orange)
![mcu](https://img.shields.io/badge/MCU-ESP32--S3-blue)
![firmware](https://img.shields.io/badge/firmware-ESP--FC-lightgrey)
![author](https://img.shields.io/badge/author-Kashi-brightgreen)

---

## Overview

A compact flight controller built from an ESP32-S3 board, an I2C gyro/accelerometer, four low-side MOSFET motor channels, and a 1S LiPo power system with a filtered 3.3 V rail. Flight control (filters, PID, mixer, arming, failsafe) comes from [ESP-FC](https://github.com/rtlopez/esp-fc) by rtlopez, which is configured with Betaflight Configurator. This repo holds the hardware design, wiring, and configuration.

**Status:** hardware design complete, firmware setup and bench testing in progress.

## Features

- ESP32-S3 running ESP-FC (Acro, Angle and Airmode, Betaflight-style tuning)
- MPU6050 gyro and accelerometer over I2C
- 4 x N-channel MOSFET drivers with flyback diodes and gate pulldowns (brushed motors)
- Receiver input: ELRS receiver over CRSF on UART2 (PPM, SBUS, IBUS also supported by ESP-FC)
- Battery voltage monitoring on an ADC pin, failsafe, optional buzzer
- Optional BMP180 barometer (not used by ESP-FC for altitude hold at the moment)

## Hardware

| Part | Qty | Notes |
|---|---|---|
| ESP32-S3 board (SuperMini or XIAO ESP32-S3) | 1 | Flight controller MCU |
| MPU6050 (GY-521) | 1 | Gyro + accelerometer, I2C 0x68 |
| BMP180 | 1 | Optional barometer, I2C 0x77 |
| ELRS receiver / Esp 32| 1 | CRSF over UART2 / Esp now|
| N-ch logic-level MOSFET | 4 | AO3400A (marking A09T), AO3416 or IRLML6344 |
| Schottky diode | 4 | SS14 (flyback) |
| Resistor 100 kohm | 4 + 2 | Gate pulldown R2, battery divider R3/R4 |
| Brushed coreless motors | 4 | 8520 class, two CW and two CCW |
| Propellers | 4 + spares | Match motor direction |
| 1S LiPo | 1 | 450 to 650 mAh, 30C or more |
| Tp 4056 | 1 | Battery charger |
| Switch, connectors, frame | - | Slide switch SW1, JST connector, 55 mm frame |

## Hardware connections

### ESP32-S3 pin map (ESP-FC settings)

| Function | ESP-FC setting | GPIO | Connects to | Notes |
|---|---|---|---|---|
| Motor 1 (output 0), rear-right | `pin_output_0` | 4 | R1 then Q1 gate | |
| Motor 2 (output 1), front-right | `pin_output_1` | 7 | R1 then Q2 gate | |
| Motor 3 (output 2), rear-left | `pin_output_2` | 3 | R1 then Q3 gate | Strapping pin, gate pulldown R2 keeps it low at boot |
| Motor 4 (output 3), front-left | `pin_output_3` | 1 | R1 then Q4 gate | |
| I2C SDA | `pin_i2c_sda` | 5 | MPU6050 SDA (BMP180 SDA) | 400 kHz |
| I2C SCL | `pin_i2c_scl` | 6 | MPU6050 SCL (BMP180 SCL) | |
| 3V3 | 3V3 pin | - | U1 output | Feed the 3V3 pin, not 5V/VIN |
| GND | GND | - | Star ground point | |

The pin map is the ESP-FC example for the XIAO ESP32-S3. On a SuperMini, check that GPIO1 to GPIO9 are broken out on your board's silkscreen before soldering. The SPI pins in the config (`pin_spi_*`) are unused with an I2C gyro; the shared GPIOs 7 and 8 are only a conflict if an SPI device is enabled.

Full CLI config: [`esp-fc-pins.cli`](esp-fc-pins.cli)

```
set pin_input_rx -1
set pin_output_0 4
set pin_output_1 7
set pin_output_2 3
set pin_output_3 1
set pin_buzzer 43
set pin_serial_0_tx 43
set pin_serial_0_rx 44
set pin_serial_1_tx 16
set pin_serial_1_rx 15
set pin_serial_2_tx 8
set pin_serial_2_rx 9
set pin_i2c_scl 6
set pin_i2c_sda 5
set pin_input_adc_0 2
set pin_input_adc_1 -1
set pin_spi_0_sck 12
set pin_spi_0_mosi 11
set pin_spi_0_miso 13
set pin_spi_cs_0 8
set pin_spi_cs_1 7
set pin_spi_cs_2 -1
set pin_buzzer_invert 1
```

### Sensor and receiver connections

| Device | Pin | Connects to |
|---|---|---|
| MPU6050 | VCC / GND | +3.3 V / GND |
| MPU6050 | SDA / SCL | GPIO5 / GPIO6 |
| MPU6050 | AD0 | GND (address 0x68) |
| BMP180 (optional) | VIN / GND | +3.3 V / GND |
| BMP180 (optional) | SDA / SCL | GPIO5 / GPIO6 (address 0x77) |


## Motor driver circuit (MOSFET + resistor + diode)

One channel, repeated four times (GPIO4, 7, 3, 1 to Q1 to Q4).

```
 VBAT_M (+) ───────────┬───────────────┐
                       │               │
                    ┌──┴──┐        ┌───┴───┐
                    │ D1  │        │   M   │  brushed motor
                    │SS14 │        │ 8520  │  (+ 100 nF across it)
                    └──┬──┘        └───┬───┘
                       └───────┬───────┘
                               │
                               │ Drain
                   R1          │
 GPIO ──────[100 ohm]───┬──────┤
                        │      │  Q1 AO3400A (N-ch, logic-level)
                        │      │
                      [R2]     │ Source
                      100k     │
                        │      │
 GND ───────────────────┴──────┴──────────────── GND
```

D1 orientation: cathode (bar) to VBAT_M, anode to the MOSFET drain.

| Part | Value | Role |
|---|---|---|
| Q1 to Q4 | AO3400A (or AO3416, IRLML6344) | Low-side switch, fully on at 3.3 V gate drive |
| R2 | 100 kohm | Holds the gate low during boot so the motor stays off |
| D1 to D4 | SS14 Schottky | Flyback diode, absorbs the motor's inductive kick |

How it works: the PWM on the gate switches the drain-source path on and off. Average motor voltage is about duty x VBAT. When Q1 turns off, the motor current circulates through D1 instead of spiking the MOSFET.

## Motor layout

ESP-FC uses Betaflight-style motor numbering (quad X, viewed from above):

```
        front
   M4 (FL)   M2 (FR)
        \   /
         \ /
         / \
        /   \
   M3 (RL)   M1 (RR)
        rear
```

| Motor | ESP-FC output | GPIO | MOSFET channel |
|---|---|---|---|
| M1 rear-right | output 0 | 4 | Q1 |
| M2 front-right | output 1 | 7 | Q2 |
| M3 rear-left | output 2 | 3 | Q3 |
| M4 front-left | output 3 | 1 | Q4 |

Spin directions follow the picture on ESP-FC's Motors tab (diagonal motors spin the same way). To reverse a motor, swap its two leads.

## Firmware setup (ESP-FC)

1. Flash ESP-FC to the ESP32-S3 (see the ESP-FC repo for the flasher or PlatformIO build).
2. Connect with the configurator, open the CLI, and paste the pin config from `esp-fc-pins.cli`, then `save`.
3. Ports tab: enable Serial Rx on UART2. Receiver tab: pick the serial receiver mode and CRSF as the provider.
4. Set the motor output protocol to brushed and the PWM rate with the `output` CLI command.
5. Motors tab: confirm each motor number and spin direction, **props off**.
6. Set gyro orientation, then check roll, pitch and yaw response in the configurator before arming.

## Input options

| Option | Extra hardware | Notes |
|---|---|---|
| ELRS receiver, CRSF on UART2 | ELRS RX | Recommended for real flights |
| ESP-NOW receiver (built into ESP-FC) | Compatible ESP-NOW transmitter | Receiver mode "SPI Rx"; see ESP-FC wireless docs |
| PPM / SBUS / IBUS | Matching receiver | Supported by ESP-FC |

## Safety

- Always test with **props off** first, then tethered, then free flight.
- Do not plug in USB while a LiPo powers the 3V3 pin. Flash with the battery disconnected.
- Test failsafe by turning the transmitter off with props removed.
- Never leave a charging or charged LiPo unattended.

## Build order

1. Build the power section and verify 3.3 V with a multimeter, no board attached.
2. Flash ESP-FC, paste the pin config, confirm the MPU6050 is detected.
3. Build one motor channel, bench test at low duty from the Motors tab.
4. Copy the channel three more times.
5. Bind the receiver, verify channels, test arming and failsafe.
6. Tune tethered, then fly.

## Planned repo structure

```
/config      ESP-FC CLI dumps and configurator backups
/hardware    Schematics, wiring photos, BOM
/docs        Notes, tuning logs
```

## Roadmap

- [ ] Hardware design and power architecture
- [ ] MOSFET motor driver design
- [ ] Pin map
- [ ] ESP-FC flashed, sensors detected
- [ ] Motor driver bench test
- [ ] Receiver binding and failsafe test
- [ ] First tethered hover and PID tuning
- [ ] Custom PCB

## Author

**Kashi** : https://kashiportfolio.vercel.app?_vercel_share=JorXNnmLWtL1gbgFv6uFBzrK2J8e7ion

Built as a hands-on embedded systems and drone project.

## Credits

Flight control firmware: [ESP-FC](https://github.com/rtlopez/esp-fc) by rtlopez (MIT licence).

