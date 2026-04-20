# Session 1.5 — FreeRTOS Intro: Tasks, Queues, Semaphores

> Week 1 | Day 5 | Duration: ~8 hours  
> Platform: ESP32 via ESP-IDF (FreeRTOS is built-in)  
> Hardware: ESP32, MPU6050, ST7920 display, buzzer, LEDs (reuse from previous sessions)  
> Prerequisite: Sessions 1.1–1.4 complete  
> Commit target: end of session

---

## Objective

Move from single-threaded, polling-based firmware to a proper RTOS task model. By the end of this session you will understand FreeRTOS tasks, inter-task communication with queues, and shared resource protection with mutexes. You will also intentionally reproduce a race condition — then fix it. This is the most important conceptual shift in Week 1.

---

## Setup (~15 min)

```bash
cd ~/embedded-training/week1
idf.py create-project session05
cd session05
idf.py set-target esp32
```

FreeRTOS is included automatically in all ESP-IDF projects. No additional setup needed.

---

## Background: Why RTOS? (read first, 20 min)

In previous sessions, your `app_main` started a few tasks with `xTaskCreate` — but you didn't think much about them. Now you need to understand what's actually happening.

**Without RTOS:** Your code runs top to bottom. If you're waiting for a UART byte, you can't also be reading a sensor. You can work around this with interrupts, but managing everything in ISRs is error-prone and doesn't scale.

**With RTOS:** Multiple "tasks" (threads) run concurrently. The scheduler switches between them based on priority and blocking state. Each task blocks on I/O, and the scheduler runs another task while it waits. This is how you write clean, modular firmware that does many things simultaneously.

**FreeRTOS key primitives:**

| Primitive | Purpose |
|-----------|---------|
| Task | Unit of execution — has its own stack |
| Queue | Pass data between tasks safely |
| Semaphore | Signal between tasks (counting or binary) |
| Mutex | Protect shared resources — only one task at a time |
| Timer | Run a callback after a delay or periodically |
| Event Group | Signal multiple events to multiple tasks |

---

## Task 1 — Tasks: Creation, Priority, and Stack (1.5 hrs)

**Goal:** Understand how tasks are created, how the scheduler works, and how priority affects execution order.

### Task creation

```c
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"

void task_a(void *arg) {
    while (1) {
        ESP_LOGI("TaskA", "Running — priority %d",
                 uxTaskPriorityGet(NULL));
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void task_b(void *arg) {
    while (1) {
        ESP_LOGI("TaskB", "Running — priority %d",
                 uxTaskPriorityGet(NULL));
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}

void app_main(void) {
    xTaskCreate(task_a, "TaskA", 2048, NULL, 5, NULL);
    xTaskCreate(task_b, "TaskB", 2048, NULL, 5, NULL);
}
```

**xTaskCreate parameters:**
```c
xTaskCreate(
    task_function,   // function pointer
    "task_name",     // name for debugging
    stack_depth,     // stack size in WORDS (not bytes) — 2048 = 8KB on 32-bit
    parameter,       // passed as void* to the function
    priority,        // 1 (low) to configMAX_PRIORITIES-1 (high)
    &handle          // optional handle for later control; NULL to ignore
);
```

### Priority experiment

```c
// Create 3 tasks at different priorities
xTaskCreate(low_task,  "Low",  2048, NULL, 3, NULL);
xTaskCreate(mid_task,  "Mid",  2048, NULL, 5, NULL);
xTaskCreate(high_task, "High", 2048, NULL, 7, NULL);
```

- Observe: when the high-priority task is ready (not blocked), it always runs first
- Make `high_task` call `vTaskDelay(pdMS_TO_TICKS(1000))` after 3 iterations — observe `mid_task` runs during that time
- Key insight: a task only gives up the CPU when it blocks (on a queue, semaphore, delay, etc.) or a higher-priority task becomes ready

### Stack monitoring

```c
void task_with_monitoring(void *arg) {
    while (1) {
        UBaseType_t watermark = uxTaskGetStackHighWaterMark(NULL);
        ESP_LOGI(TAG, "Stack high watermark: %u words remaining", watermark);
        vTaskDelay(pdMS_TO_TICKS(2000));
    }
}
```

- Stack overflow kills everything — often silently
- `uxTaskGetStackHighWaterMark` returns minimum free stack ever recorded
- If it's close to 0, increase stack size in `xTaskCreate`
- Rule of thumb: start with 4096 bytes (2048 words) and monitor

### Task list

```c
// Print all running tasks (requires configUSE_TRACE_FACILITY=1 in sdkconfig)
void print_task_list(void) {
    char buf[512];
    vTaskList(buf);
    printf("Name          State  Pri  Stack  Num\n");
    printf("%s", buf);
}
```

Enable in `sdkconfig`:
```
CONFIG_FREERTOS_USE_TRACE_FACILITY=y
CONFIG_FREERTOS_USE_STATS_FORMATTING_FUNCTIONS=y
```

---

## Task 2 — Queues: Producer/Consumer Pattern (2 hrs)

**Goal:** Pass sensor data from a reader task to a display task using a queue. This is the fundamental inter-task communication pattern.

### Queue basics

```c
#include "freertos/queue.h"

// Create a queue: 10 items, each sizeof(SensorData)
QueueHandle_t sensor_queue = xQueueCreate(10, sizeof(SensorData));

// Producer: send to queue (from a task — not ISR)
xQueueSend(sensor_queue, &data, pdMS_TO_TICKS(10));  // 10ms timeout

// Consumer: receive from queue
SensorData received;
xQueueReceive(sensor_queue, &received, portMAX_DELAY);  // block until data arrives
```

### Multi-sensor pipeline

Build a pipeline with three tasks:

```
[MPU6050 Reader Task] ──queue──► [Data Processor Task] ──queue──► [Display Task]
```

```c
typedef struct {
    float ax, ay, az;
    float gx, gy, gz;
    float temp;
    uint32_t timestamp_ms;
} IMUData;

typedef struct {
    float pitch_deg;
    float roll_deg;
    float temp;
    uint32_t timestamp_ms;
} ProcessedData;
```

**Reader task** (highest priority — reads hardware):
```c
void imu_reader_task(void *arg) {
    QueueHandle_t q = (QueueHandle_t)arg;
    mpu6050_wake();

    while (1) {
        MPU6050Raw raw;
        mpu6050_read_raw(&raw);
        MPU6050Data mpu = mpu6050_convert(&raw);

        IMUData data = {
            .ax = mpu.ax_g, .ay = mpu.ay_g, .az = mpu.az_g,
            .gx = mpu.gx_dps, .gy = mpu.gy_dps, .gz = mpu.gz_dps,
            .temp = mpu.temp_c,
            .timestamp_ms = esp_timer_get_time() / 1000,
        };

        if (xQueueSend(q, &data, pdMS_TO_TICKS(10)) != pdTRUE) {
            ESP_LOGW(TAG, "Reader: queue full, dropping sample");
        }
        vTaskDelay(pdMS_TO_TICKS(50));  // 20Hz sample rate
    }
}
```

**Processor task** (medium priority — computes pitch/roll):
```c
// Pitch and roll from accelerometer (simple tilt sensing, no gyro fusion)
float compute_pitch(float ax, float ay, float az) {
    return atan2f(ax, sqrtf(ay * ay + az * az)) * 180.0f / M_PI;
}

float compute_roll(float ax, float ay, float az) {
    return atan2f(ay, sqrtf(ax * ax + az * az)) * 180.0f / M_PI;
}

void processor_task(void *arg) {
    // arg is a struct with both queues
    TaskQueues *qs = (TaskQueues *)arg;
    IMUData raw;

    while (1) {
        if (xQueueReceive(qs->raw_q, &raw, portMAX_DELAY)) {
            ProcessedData out = {
                .pitch_deg    = compute_pitch(raw.ax, raw.ay, raw.az),
                .roll_deg     = compute_roll(raw.ax, raw.ay, raw.az),
                .temp         = raw.temp,
                .timestamp_ms = raw.timestamp_ms,
            };
            xQueueSend(qs->proc_q, &out, pdMS_TO_TICKS(10));
        }
    }
}
```

**Display task** (lowest priority — output only):
```c
void display_task(void *arg) {
    QueueHandle_t q = (QueueHandle_t)arg;
    ProcessedData d;
    char line[17];

    while (1) {
        if (xQueueReceive(q, &d, portMAX_DELAY)) {
            snprintf(line, sizeof(line), "P:%+6.1f", d.pitch_deg);
            st7920_print(0, 0, line);
            snprintf(line, sizeof(line), "R:%+6.1f", d.roll_deg);
            st7920_print(1, 0, line);
            ESP_LOGI(TAG, "[%lu ms] Pitch: %.1f  Roll: %.1f  Temp: %.1f C",
                     d.timestamp_ms, d.pitch_deg, d.roll_deg, d.temp);
        }
    }
}
```

**Understand:**
- What happens when the queue is full and the producer drops a sample?
- Why does the display task have the lowest priority?
- What would happen if `portMAX_DELAY` was replaced with `0` in `xQueueReceive`?

---

## Task 3 — Mutexes: Protecting Shared State (1.5 hrs)

**Goal:** Reproduce a race condition on shared UART access, then fix it with a mutex.

### The problem: unprotected shared resource

Run two tasks that both print to UART without synchronization:

```c
void spammer_a(void *arg) {
    while (1) {
        uart_send_str("AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAA\r\n");
        vTaskDelay(1);   // yield but only for 1 tick
    }
}

void spammer_b(void *arg) {
    while (1) {
        uart_send_str("BBBBBBBBBBBBBBBBBBBBBBBBBBBBBBB\r\n");
        vTaskDelay(1);
    }
}
```

Run both tasks at the same priority. Observe the output — you will see interleaved garbage like:
```
AAAAAABBBBBBBAAAAAABBBBBBBBB
```

This is the race condition. The scheduler switches tasks mid-write.

### The fix: mutex

```c
#include "freertos/semphr.h"

SemaphoreHandle_t uart_mutex;

void protected_uart_send(const char *str) {
    if (xSemaphoreTake(uart_mutex, pdMS_TO_TICKS(100)) == pdTRUE) {
        uart_send_str(str);
        xSemaphoreGive(uart_mutex);
    } else {
        // timeout — handle the error
    }
}

void app_main(void) {
    uart_mutex = xSemaphoreCreateMutex();
    // ...
}
```

Rerun with `protected_uart_send` — output is now clean.

### Mutex rules

- `xSemaphoreTake` — acquires the mutex; blocks if another task holds it
- `xSemaphoreGive` — releases the mutex
- Never call `xSemaphoreTake` from an ISR — use `xSemaphoreTakeFromISR`
- Always release a mutex you've taken — even on error paths (this is where RAII from Week 0 would help)
- Never hold a mutex while blocking on I/O — deadlock risk

### Apply mutex to your sensor pipeline

Protect SPI bus access so both the BMP280 and ST7920 tasks can share it safely:

```c
SemaphoreHandle_t spi_mutex;

void bmp280_task(void *arg) {
    while (1) {
        if (xSemaphoreTake(spi_mutex, portMAX_DELAY)) {
            BMP280Data d = bmp280_read(&cal);
            xSemaphoreGive(spi_mutex);
            xQueueSend(sensor_queue, &d, 0);
        }
        vTaskDelay(pdMS_TO_TICKS(500));
    }
}
```

---

## Task 4 — Semaphores: Task Signaling (1 hr)

**Goal:** Use a binary semaphore to signal between an ISR and a task. This is the correct pattern for interrupt-driven sensor reads.

### Binary semaphore vs mutex

- **Mutex:** for mutual exclusion — one task at a time owns it. Has priority inheritance.
- **Binary semaphore:** for signaling — one task signals another. No ownership concept.

### PIR motion → semaphore → task

```c
SemaphoreHandle_t motion_sem;

static void IRAM_ATTR pir_isr(void *arg) {
    BaseType_t higher_priority_woken = pdFALSE;
    xSemaphoreGiveFromISR(motion_sem, &higher_priority_woken);
    portYIELD_FROM_ISR(higher_priority_woken);
}

void motion_handler_task(void *arg) {
    while (1) {
        if (xSemaphoreTake(motion_sem, portMAX_DELAY)) {
            ESP_LOGI(TAG, "Motion detected — triggering alarm");
            buzzer_set_freq(880);
            vTaskDelay(pdMS_TO_TICKS(500));
            buzzer_stop();
        }
    }
}

void app_main(void) {
    motion_sem = xSemaphoreCreateBinary();
    gpio_isr_handler_add(PIR_PIN, pir_isr, NULL);
    xTaskCreate(motion_handler_task, "motion", 2048, NULL, 8, NULL);
}
```

**`portYIELD_FROM_ISR`:** If giving the semaphore unblocks a higher-priority task, this causes an immediate context switch at the end of the ISR — rather than waiting for the next scheduler tick. Use it in every ISR that unblocks a task.

---

## Task 5 — Break It and Fix It: Priority Inversion Preview (1 hr)

**Goal:** Understand the concept of priority inversion. You'll go deep on this in Week 2 — today just experience it.

### Setup: 3 tasks, 1 mutex

```
High priority task H — needs mutex M to run
Medium priority task M — runs for a long time, never blocks
Low priority task L — holds mutex M
```

```c
SemaphoreHandle_t shared_mutex;

void high_task(void *arg) {
    vTaskDelay(pdMS_TO_TICKS(100));  // give L time to grab mutex
    ESP_LOGI(TAG, "H: waiting for mutex");
    uint32_t t0 = esp_timer_get_time() / 1000;
    xSemaphoreTake(shared_mutex, portMAX_DELAY);
    uint32_t t1 = esp_timer_get_time() / 1000;
    ESP_LOGI(TAG, "H: got mutex after %lu ms", t1 - t0);
    xSemaphoreGive(shared_mutex);
    vTaskDelete(NULL);
}

void medium_task(void *arg) {
    vTaskDelay(pdMS_TO_TICKS(50));   // start after L grabs mutex
    ESP_LOGI(TAG, "M: running (CPU hog)");
    // Busy loop — no blocking, no yielding
    volatile uint32_t count = 0;
    for (uint32_t i = 0; i < 10000000; i++) count++;
    ESP_LOGI(TAG, "M: done");
    vTaskDelete(NULL);
}

void low_task(void *arg) {
    xSemaphoreTake(shared_mutex, portMAX_DELAY);
    ESP_LOGI(TAG, "L: holding mutex, doing work");
    vTaskDelay(pdMS_TO_TICKS(2000));   // holds mutex for 2 seconds
    ESP_LOGI(TAG, "L: releasing mutex");
    xSemaphoreGive(shared_mutex);
    vTaskDelete(NULL);
}

void app_main(void) {
    shared_mutex = xSemaphoreCreateMutex();  // plain mutex, no priority inheritance
    xTaskCreate(low_task,    "Low",    2048, NULL, 3, NULL);
    xTaskCreate(medium_task, "Medium", 2048, NULL, 5, NULL);
    xTaskCreate(high_task,   "High",   2048, NULL, 7, NULL);
}
```

**Observe:** H waits far longer than the 2 seconds L holds the mutex — because M preempts L and L never gets CPU time to release the mutex. H is effectively blocked by M, even though M has lower priority than H. That's priority inversion.

**Fix:** Use `xSemaphoreCreateMutex()` (which has priority inheritance in FreeRTOS) and observe H no longer waits as long. Document the timing difference in your `NOTES.md`.

---

## Wrap-up Checklist

Before committing:

- [ ] 3-task priority experiment shows high-priority task preempting lower ones
- [ ] Stack watermark monitoring implemented and printing reasonable values
- [ ] Producer/consumer pipeline: IMU → processor → display working end-to-end
- [ ] Race condition on UART reproduced (screenshot or log snippet in NOTES)
- [ ] Mutex fixes the race condition — clean UART output confirmed
- [ ] PIR ISR → semaphore → task pattern working (motion triggers buzzer)
- [ ] Priority inversion scenario reproduced and timing documented
- [ ] Add `NOTES.md`: timing numbers from priority inversion, what `portYIELD_FROM_ISR` does and why it matters

---

## Commit

```bash
git add week1/session05/
git commit -m "week1/session05: FreeRTOS tasks, queues, mutexes, semaphores, priority inversion demo"
git push
```

---

## Week 1 Complete — Review Before Week 2

You now have hands-on experience with:
- ESP-IDF project structure and toolchain
- GPIO, interrupts, timers, PWM, ADC
- UART with event-driven RX and command parser
- SPI: BMP280 sensor + ST7920 display, raw register access
- I2C: MPU6050 + DS3231, raw register access, BCD encoding
- FreeRTOS: tasks, queues, mutexes, binary semaphores, priority inversion

**Check before moving to Week 2:**
- [ ] All 5 sessions committed to GitHub with meaningful commit messages
- [ ] Every session has a `NOTES.md`
- [ ] You can explain out loud: why ISRs post to queues, what a mutex protects, and what priority inversion is
- [ ] Hardware still works — no fried pins

Week 2 goes deeper on FreeRTOS and introduces Linux kernel drivers on the RPi.

---

## What to Read This Weekend (optional, 2 hrs each day)

**Saturday:** FreeRTOS internals — [FreeRTOS documentation](https://www.freertos.org/Documentation/RTOS_book.html) chapters on tasks and queues. Focus on the scheduler, task states (Running/Ready/Blocked/Suspended), and how `vTaskDelay` works internally.

**Sunday:** Review your Week 1 code as if you're an interviewer. Can you explain every line? Start skimming the Zephyr getting started docs — you'll need them in Week 3.
