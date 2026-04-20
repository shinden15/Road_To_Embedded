# Week 1 — Bare-metal C + Peripherals on ESP32

> Platform: ESP32 via ESP-IDF  
> Sessions: 5 weekdays × 8 hrs + optional weekend reading (2 hrs/day)  
> Hardware needed: ESP32, breadboard, jumper wires, buzzer, LEDs, buttons, PIR sensor, potentiometer, BMP280, GME12864-11, MPU6050, DS3231

---

## Goals

- Get comfortable with ESP-IDF project structure and toolchain
- Drive real hardware at the register/HAL level — no Arduino abstractions
- Implement SPI and I2C from scratch using datasheets
- Establish the habit of reading datasheets before writing code

---

## Setup Checklist (do before Session 1.1)

- [ ] Install ESP-IDF (latest stable): https://docs.espressif.com/projects/esp-idf/en/latest/esp32/get-started/
- [ ] Verify `idf.py build`, `idf.py flash`, `idf.py monitor` work on a hello-world project
- [ ] Create GitHub repo: `embedded-training` — this is where all weeks live
- [ ] Create folder structure: `week1/session1/`, `week1/session2/`, etc.

---

## Sessions

### Session 1.1 — ESP-IDF Setup, GPIO, Interrupts, Timers, PWM
**Focus:** The building blocks of every embedded project

Topics:
- ESP-IDF project structure: `CMakeLists.txt`, `sdkconfig`, `main/`
- GPIO: output (LED), input (button, PIR), pull-up/pull-down config
- GPIO interrupts: ISR registration, `IRAM_ATTR`, debouncing
- Hardware timers: periodic callbacks
- LEDC peripheral for PWM — drive a buzzer or fade an LED

**Deliverable:**
- Blink an LED using a hardware timer (not `vTaskDelay`)
- Button press triggers GPIO interrupt — toggles LED state
- PIR motion sensor triggers interrupt — prints event to serial
- Potentiometer read via ADC — prints value to serial
- Buzzer driven via PWM at varying frequencies

---

### Session 1.2 — UART
**Focus:** Serial communication — the most fundamental debug and comms tool

Topics:
- UART configuration: baud rate, parity, stop bits
- ESP-IDF UART driver API
- TX/RX buffering
- Polling vs interrupt-driven receive
- Building a command parser over UART

**Deliverable:**
- ESP32 receives commands from PC over UART (via `idf.py monitor` or `minicom`)
- Implement at least 3 commands: e.g. `LED ON`, `LED OFF`, `ADC READ`
- Respond with status over UART

---

### Session 1.3 — SPI
**Focus:** Synchronous serial protocol — used for displays, flash, sensors

Topics:
- SPI fundamentals: MOSI, MISO, SCLK, CS; 4 modes (CPOL/CPHA)
- ESP-IDF SPI master driver
- Reading the BMP280/BMP580 datasheet — find register map, read/write protocol
- Reading the ST7920 (GME12864-11) datasheet — initialization sequence

**Deliverable:**
- Initialize and read temperature/pressure from BMP280 via SPI
- Initialize GME12864-11 display and render text or a pattern
- All SPI config done manually — understand every parameter you set

---

### Session 1.4 — I2C
**Focus:** Multi-device bus protocol — used for sensors, RTCs, EEPROMs

Topics:
- I2C fundamentals: SDA, SCL, addressing, ACK/NACK, clock stretching
- ESP-IDF I2C master driver
- Reading the MPU6050 datasheet — WHO_AM_I register, accel/gyro data registers
- Reading the DS3231 datasheet — timekeeping registers, BCD encoding

**Deliverable:**
- Read accelerometer + gyroscope data from MPU6050 via I2C
- Read current time from DS3231 via I2C
- Print both to serial in a readable format
- No third-party sensor libraries — raw register reads only

---

### Session 1.5 — FreeRTOS Intro: Tasks, Queues, Semaphores
**Focus:** Moving from bare-metal polling to an RTOS task model

Topics:
- FreeRTOS basics: tasks, scheduler, priorities
- `xTaskCreate`, `vTaskDelay`, `vTaskDelete`
- Queues: passing data between tasks
- Semaphores and mutexes: protecting shared state
- Why you need synchronization (race conditions on shared peripherals)

**Deliverable:**
- Two tasks: one reads MPU6050 data, one prints it to serial
- Data passed between tasks via a queue
- Shared UART access protected by a mutex
- Demonstrate what happens without the mutex (intentionally break it, then fix it)

---

## Weekend Reading

| Day | Topic |
|-----|-------|
| Sat | FreeRTOS internals: how the scheduler works, priority inversion, task states |
| Sun | Review Week 1 code; read ESP-IDF docs for any peripheral you found confusing |

---

## Ground Rules

- No Arduino libraries — ESP-IDF APIs only
- Read the datasheet before writing code for any sensor or peripheral
- Commit every session — one commit per session minimum
- When something breaks, document it in your commit message: what broke, how you found it, how you fixed it
