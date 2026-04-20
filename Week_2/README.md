# Week 2 — FreeRTOS Deep Dive + Linux Kernel Drivers

> Platforms: ESP32 (FreeRTOS), Raspberry Pi (Linux kernel)  
> Sessions: 5 weekdays × 8 hrs + optional weekend reading (2 hrs/day)  
> Hardware needed: ESP32 ×2, nRF24L01 ×2, RPi, MPU6050, BMP280

---

## Goals

- Deeply understand FreeRTOS scheduling, synchronization, and common failure modes
- Write a real Linux kernel module and character device driver
- Use the Linux I2C and SPI kernel subsystems to talk to hardware
- Understand device trees at a basic level

---

## Sessions

### Session 2.1 — FreeRTOS: Timers, Priority Inversion, Debugging
**Focus:** The parts of RTOS that trip people up in production

Topics:
- Software timers vs hardware timers in FreeRTOS
- Priority inversion: what it is, why it causes deadlocks
- Priority inheritance: how FreeRTOS mutexes handle it
- `configASSERT`, `uxTaskGetStackHighWaterMark`, `vTaskList`
- Stack overflow detection

**Deliverable:**
- Intentionally reproduce a priority inversion scenario with 3 tasks
- Fix it using a mutex with priority inheritance
- Write up what happened, how you detected it, and how you fixed it (in README)

---

### Session 2.2 — FreeRTOS + nRF24L01 Wireless
**Focus:** Combining SPI peripheral management with RTOS task design

Topics:
- nRF24L01 protocol: PTX/PRX modes, pipes, auto-ACK
- SPI access from multiple tasks — why you need a mutex
- Designing a clean task architecture for a wireless sensor node

**Deliverable:**
- ESP32 #1 (transmitter): reads MPU6050 data, sends over nRF24L01
- ESP32 #2 (receiver): receives data, prints to serial
- Each ESP32 uses at least 2 FreeRTOS tasks with proper synchronization
- SPI bus access protected by mutex

---

### Session 2.3 — Linux Kernel Module Basics on RPi
**Focus:** Writing code that runs in kernel space

Topics:
- Kernel space vs user space
- Kernel module structure: `module_init`, `module_exit`, `MODULE_LICENSE`
- `printk` and kernel log levels
- Character devices: `file_operations`, `register_chrdev`
- `insmod`, `rmmod`, `lsmod`, `dmesg`

**Deliverable:**
- Write a kernel module that registers a character device
- Writing to the device echoes data back when read
- Load and unload cleanly with `insmod`/`rmmod`
- No kernel panics

---

### Session 2.4 — Linux I2C Subsystem + Device Tree
**Focus:** The proper kernel way to talk to I2C hardware

Topics:
- Linux I2C subsystem: `i2c_driver`, `i2c_client`, `i2c_transfer`
- Device tree basics: what it is, why Linux uses it
- Writing a minimal device tree overlay for RPi
- Binding a kernel driver to a device tree node
- `i2c-tools`: `i2cdetect`, `i2cdump`, `i2cget`

**Deliverable:**
- Write a kernel I2C driver for MPU6050 on RPi
- Read accelerometer data from kernel driver; expose via `/dev` or `sysfs`
- Device tree overlay binds the driver to the correct I2C bus and address

---

### Session 2.5 — Linux SPI Subsystem + Device Tree
**Focus:** SPI in the kernel — same concepts, different subsystem

Topics:
- Linux SPI subsystem: `spi_driver`, `spi_device`, `spi_transfer`, `spi_message`
- SPI device tree configuration on RPi
- Differences between I2C and SPI kernel driver structure

**Deliverable:**
- Write a kernel SPI driver for BMP280 on RPi
- Read temperature/pressure; expose via `sysfs`
- Compare the driver structure to your I2C driver — document the differences

---

## Weekend Reading

| Day | Topic |
|-----|-------|
| Sat | Linux Device Drivers (LDD3) Ch. 1–3 (free online); Linux kernel driver model overview |
| Sun | Review Week 2 code; skim Zephyr getting started guide and architecture overview |

---

## Ground Rules

- No Arduino libraries — ESP-IDF only on ESP32
- On RPi: work in kernel space — avoid user-space I2C/SPI libs like `smbus`
- Commit every session
- Document every kernel panic or build error in your commit message
