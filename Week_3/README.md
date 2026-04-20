# Week 3 — Zephyr RTOS + Hardware Debugging

> Platforms: QEMU (Zephyr), ESP32 (Zephyr), ESP32 (logic analyzer + JTAG)  
> Sessions: 5 weekdays × 8 hrs + optional weekend reading (2 hrs/day)  
> Hardware needed: ESP32, MPU6050, BMP280, nRF24L01 ×2, logic analyzer, ST-Link V2

---

## Goals

- Get productive with Zephyr: build system, device model, Kconfig, DTS
- Port previous ESP-IDF work into Zephyr to understand the differences
- Use a logic analyzer to capture and decode real SPI/I2C traffic
- Set up and use JTAG debugging with OpenOCD on ESP32

---

## Setup Checklist (do before Session 3.1)

- [ ] Install Zephyr SDK and `west`: https://docs.zephyrproject.org/latest/develop/getting_started/
- [ ] Verify QEMU target builds and runs: `west build -b qemu_x86 samples/hello_world`
- [ ] Install PulseView for logic analyzer
- [ ] Install OpenOCD

---

## Sessions

### Session 3.1 — Zephyr Setup, QEMU, Kconfig, Device Tree
**Focus:** Understanding Zephyr's build system and configuration model

Topics:
- Zephyr project structure: `west`, `CMakeLists.txt`, `prj.conf`, `.overlay`
- Kconfig: enabling/disabling features at build time
- Device tree in Zephyr: how hardware is described, how drivers bind to nodes
- QEMU as a development target — no hardware needed
- Zephyr shell and logging subsystem

**Deliverable:**
- Blink sample running on QEMU
- Serial output working via Zephyr logging API
- Understand and modify at least one Kconfig option and one DTS node

---

### Session 3.2 — Zephyr GPIO, UART, Device Model
**Focus:** Driving hardware through Zephyr's abstraction layer

Topics:
- Zephyr device model: `DEVICE_DT_GET`, `device_is_ready`
- GPIO API: `gpio_pin_configure`, `gpio_pin_set`, `gpio_pin_interrupt_configure`
- UART API: `uart_tx`, `uart_rx_enable`, callbacks
- Zephyr kernel primitives: threads, semaphores, message queues

**Deliverable:**
- Port Session 1.1 (GPIO/PWM/interrupt) and Session 1.2 (UART command parser) into Zephyr on ESP32
- Note every place where the Zephyr API differs from ESP-IDF — document it

---

### Session 3.3 — Zephyr I2C/SPI Drivers + Sensor API
**Focus:** Using Zephyr's sensor abstraction to read hardware

Topics:
- Zephyr I2C API: `i2c_write_read`, `i2c_burst_read`
- Zephyr SPI API: `spi_transceive`
- Zephyr sensor API: `sensor_sample_fetch`, `sensor_channel_get`
- Writing a minimal Zephyr out-of-tree driver

**Deliverable:**
- Read MPU6050 data via Zephyr I2C API
- Read BMP280 data via Zephyr SPI API
- If time allows: write a minimal out-of-tree Zephyr driver for DS3231

---

### Session 3.4 — Logic Analyzer + JTAG/OpenOCD Debugging
**Focus:** Hardware debugging — seeing what's actually on the wire

Topics:
- Logic analyzer setup with PulseView
- Capturing and decoding SPI traffic (clock, CS, MOSI, MISO)
- Capturing and decoding I2C traffic (SDA, SCL)
- JTAG on ESP32: built-in USB JTAG interface
- OpenOCD: connecting, halting, stepping, inspecting registers
- GDB + OpenOCD for source-level debugging

**Deliverable:**
- Capture SPI transaction to BMP280 — decode and verify against datasheet
- Capture I2C transaction to MPU6050 — identify address, register, data bytes
- Set a breakpoint in your Zephyr code via GDB + OpenOCD; inspect a variable

---

### Session 3.5 — Zephyr + nRF24L01 Wireless
**Focus:** Porting the FreeRTOS wireless project to Zephyr

Topics:
- SPI in Zephyr with chip select handling
- Zephyr threads vs FreeRTOS tasks — what's different
- Zephyr work queues

**Deliverable:**
- Port Session 2.2 (nRF24L01 wireless) into Zephyr
- ESP32 #1 transmits MPU6050 data; ESP32 #2 receives and prints
- Document: what was easier in Zephyr vs ESP-IDF, what was harder

---

## Weekend Reading

| Day | Topic |
|-----|-------|
| Sat | Zephyr architecture deep dive; browse ADI's Zephyr driver contributions on GitHub |
| Sun | Browse open Zephyr GitHub issues tagged `good first issue`; read ADI product portfolio overview |

---

## Ground Rules

- Commit every session
- For every Zephyr session: document what's different from ESP-IDF in your README
- Logic analyzer captures should be saved and included in your GitHub repo (PulseView `.sr` files)
