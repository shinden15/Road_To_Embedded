# Session 1.1 — ESP-IDF Setup, GPIO, Interrupts, Timers, PWM

> Week 1 | Day 1 | Duration: ~8 hours  
> Platform: ESP32 via ESP-IDF  
> Hardware: ESP32, LEDs, buttons, PIR sensor (HC-SR501), potentiometer, active/passive buzzer, jumper wires, breadboard  
> Commit target: end of session

---

## Objective

Get ESP-IDF fully working and drive real hardware without any Arduino abstractions. By the end of this session you will have configured GPIO, handled interrupts, driven PWM, and read analog values — all through ESP-IDF APIs directly. Every parameter you set should be something you can explain.

---

## Setup (~1 hr)

### Install ESP-IDF

```bash
# Linux/macOS
git clone --recursive https://github.com/espressif/esp-idf.git ~/esp/esp-idf
cd ~/esp/esp-idf
git checkout v5.1  # latest stable at time of writing — check https://github.com/espressif/esp-idf/releases
./install.sh esp32
source export.sh   # add to your ~/.bashrc or ~/.zshrc
```

### Verify installation

```bash
idf.py --version
```

### Create project

```bash
mkdir -p ~/embedded-training/week1/session01
cd ~/embedded-training/week1/session01
idf.py create-project session01
cd session01
idf.py set-target esp32
```

### Verify build + flash pipeline

```bash
idf.py build
idf.py -p /dev/ttyUSB0 flash   # adjust port: Windows = COM3, macOS = /dev/cu.usbserial-*
idf.py -p /dev/ttyUSB0 monitor
```

- [ ] Build succeeds
- [ ] Flash succeeds
- [ ] Monitor shows output (`Ctrl+]` to exit monitor)

### GitHub

```bash
cd ~/embedded-training
git init          # if not already done
git remote add origin <your-repo-url>
```

---

## Wiring Reference

| Component | ESP32 Pin | Notes |
|-----------|-----------|-------|
| LED | GPIO2 | Built-in LED on most devkits — no resistor needed |
| External LED | GPIO4 | Add 220Ω resistor in series |
| Button | GPIO0 | Already on devkit as BOOT button; or wire to any GPIO with pull-up |
| PIR (HC-SR501) VCC | 5V | PIR needs 5V supply |
| PIR (HC-SR501) GND | GND | |
| PIR (HC-SR501) OUT | GPIO5 | 3.3V-compatible output |
| Potentiometer VCC | 3.3V | Do NOT use 5V on ADC pins |
| Potentiometer GND | GND | |
| Potentiometer WIPER | GPIO34 | ADC1_CH6 — input only pin |
| Active Buzzer + | GPIO18 | Drive HIGH to activate |
| Active Buzzer - | GND | |
| Passive Buzzer + | GPIO19 | Drive with PWM |
| Passive Buzzer - | GND | |

> **Important:** ESP32 ADC pins max 3.3V. Never connect 5V to an ADC pin.

---

## Task 1 — GPIO Output: Blink Without Arduino (45 min)

**Goal:** Blink the LED using a hardware timer — not `vTaskDelay`, not `HAL_Delay`. Understand every line of the configuration.

### Implementation

```c
#include "driver/gpio.h"
#include "esp_timer.h"
#include "esp_log.h"

#define LED_PIN GPIO_NUM_2
static const char *TAG = "session01";

static void led_timer_cb(void *arg) {
    static uint8_t state = 0;
    gpio_set_level(LED_PIN, state ^= 1);
}

void app_main(void) {
    // Configure GPIO
    gpio_config_t led_cfg = {
        .pin_bit_mask = (1ULL << LED_PIN),
        .mode         = GPIO_MODE_OUTPUT,
        .pull_up_en   = GPIO_PULLUP_DISABLE,
        .pull_down_en = GPIO_PULLDOWN_DISABLE,
        .intr_type    = GPIO_INTR_DISABLE,
    };
    gpio_config(&led_cfg);

    // Configure periodic timer
    esp_timer_handle_t timer;
    esp_timer_create_args_t timer_args = {
        .callback = led_timer_cb,
        .name     = "led_blink",
    };
    esp_timer_create(&timer_args, &timer);
    esp_timer_start_periodic(timer, 500000);  // 500ms in microseconds

    ESP_LOGI(TAG, "Blink started");
}
```

**Understand before moving on:**
- What does `pin_bit_mask` mean? Why is it `1ULL << pin`?
- What unit does `esp_timer_start_periodic` use? What is 500000?
- Why `static uint8_t state` inside the callback?

---

## Task 2 — GPIO Input + Interrupt: Button and PIR (1.5 hrs)

**Goal:** Handle GPIO interrupts correctly — including the ISR constraints that trip people up.

### Key concepts before coding

- ISR (Interrupt Service Routine) runs in interrupt context — very restricted
- Cannot call most ESP-IDF functions from an ISR
- Cannot use `printf` or `ESP_LOGI` from an ISR
- Safe pattern: set a flag or post to a FreeRTOS queue from ISR; handle in a task
- `IRAM_ATTR` — places the ISR in IRAM so it can execute even when flash cache is disabled

### Implementation

```c
#include "driver/gpio.h"
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/queue.h"

#define BUTTON_PIN GPIO_NUM_0
#define PIR_PIN    GPIO_NUM_5

static QueueHandle_t gpio_event_queue;

typedef struct {
    uint32_t pin;
    uint8_t  level;
} GpioEvent;

static void IRAM_ATTR gpio_isr_handler(void *arg) {
    GpioEvent evt = {
        .pin   = (uint32_t)arg,
        .level = gpio_get_level((gpio_num_t)(uint32_t)arg),
    };
    xQueueSendFromISR(gpio_event_queue, &evt, NULL);
}

static void gpio_task(void *arg) {
    GpioEvent evt;
    while (1) {
        if (xQueueReceive(gpio_event_queue, &evt, portMAX_DELAY)) {
            if (evt.pin == BUTTON_PIN) {
                ESP_LOGI(TAG, "Button %s", evt.level ? "released" : "pressed");
                gpio_set_level(LED_PIN, !evt.level);
            } else if (evt.pin == PIR_PIN) {
                ESP_LOGI(TAG, "PIR: %s", evt.level ? "motion detected" : "clear");
            }
        }
    }
}
```

**Configure both pins:**
```c
// Button: input, pull-up, interrupt on any edge
gpio_config_t btn_cfg = {
    .pin_bit_mask = (1ULL << BUTTON_PIN),
    .mode         = GPIO_MODE_INPUT,
    .pull_up_en   = GPIO_PULLUP_ENABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type    = GPIO_INTR_ANYEDGE,
};
gpio_config(&btn_cfg);

// PIR: input, no pull (has its own output driver), rising edge
gpio_config_t pir_cfg = {
    .pin_bit_mask = (1ULL << PIR_PIN),
    .mode         = GPIO_MODE_INPUT,
    .pull_up_en   = GPIO_PULLUP_DISABLE,
    .pull_down_en = GPIO_PULLDOWN_DISABLE,
    .intr_type    = GPIO_INTR_POSEDGE,
};
gpio_config(&pir_cfg);

// Install ISR service and hook handlers
gpio_install_isr_service(0);
gpio_isr_handler_add(BUTTON_PIN, gpio_isr_handler, (void *)BUTTON_PIN);
gpio_isr_handler_add(PIR_PIN,    gpio_isr_handler, (void *)PIR_PIN);
```

**Understand before moving on:**
- Why can't you call `ESP_LOGI` directly from `gpio_isr_handler`?
- What does `IRAM_ATTR` do and when is it required?
- Why use a queue instead of a global flag?
- What is `portMAX_DELAY`?

### Debouncing

Mechanical buttons bounce — a single press generates multiple edges. Add software debouncing:

```c
static void IRAM_ATTR gpio_isr_handler(void *arg) {
    static uint32_t last_time = 0;
    uint32_t now = esp_timer_get_time() / 1000;  // ms
    if ((now - last_time) < 50) return;           // 50ms debounce
    last_time = now;

    GpioEvent evt = { .pin = (uint32_t)arg };
    xQueueSendFromISR(gpio_event_queue, &evt, NULL);
}
```

> **Note:** Calling `esp_timer_get_time()` from ISR is safe — it reads a hardware register directly.

---

## Task 3 — ADC: Read Potentiometer (45 min)

**Goal:** Read analog values from the potentiometer. Understand ESP32 ADC limitations.

### ESP32 ADC notes (important)

- ESP32 ADC is 12-bit (0–4095)
- ADC accuracy is poor near 0V and 3.3V — usable range is roughly 100mV–3.1V
- ADC2 cannot be used when Wi-Fi is active — use ADC1 pins (GPIO32–GPIO39)
- GPIO34–GPIO39 are input-only (no pull-up/pull-down)

### Implementation

```c
#include "esp_adc/adc_oneshot.h"
#include "esp_adc/adc_cali.h"
#include "esp_adc/adc_cali_scheme.h"

#define POT_CHANNEL ADC_CHANNEL_6   // GPIO34

adc_oneshot_unit_handle_t adc_handle;
adc_cali_handle_t         cali_handle;

void adc_init(void) {
    // Init ADC unit
    adc_oneshot_unit_init_cfg_t unit_cfg = {
        .unit_id = ADC_UNIT_1,
    };
    adc_oneshot_new_unit(&unit_cfg, &adc_handle);

    // Configure channel
    adc_oneshot_chan_cfg_t chan_cfg = {
        .atten    = ADC_ATTEN_DB_12,   // 0–3.3V range
        .bitwidth = ADC_BITWIDTH_12,
    };
    adc_oneshot_config_channel(adc_handle, POT_CHANNEL, &chan_cfg);

    // Optional: calibration for voltage reading
    adc_cali_line_fitting_config_t cali_cfg = {
        .unit_id  = ADC_UNIT_1,
        .atten    = ADC_ATTEN_DB_12,
        .bitwidth = ADC_BITWIDTH_12,
    };
    adc_cali_create_scheme_line_fitting(&cali_cfg, &cali_handle);
}

void adc_read_task(void *arg) {
    int raw, voltage_mv;
    while (1) {
        adc_oneshot_read(adc_handle, POT_CHANNEL, &raw);
        adc_cali_raw_to_voltage(cali_handle, raw, &voltage_mv);
        ESP_LOGI(TAG, "ADC raw: %d  voltage: %d mV", raw, voltage_mv);
        vTaskDelay(pdMS_TO_TICKS(200));
    }
}
```

**Understand before moving on:**
- What is `ADC_ATTEN_DB_12`? What does attenuation affect?
- Why is calibration necessary?
- What is the raw value range and how does it map to voltage?

---

## Task 4 — PWM: Drive Buzzer and Fade LED (1.5 hrs)

**Goal:** Use the LEDC peripheral to generate PWM. Control frequency (for the passive buzzer) and duty cycle (for LED brightness).

### ESP32 PWM notes

- ESP32 has a dedicated LEDC (LED Controller) peripheral for PWM
- 8 high-speed + 8 low-speed channels
- Configurable frequency and resolution (duty cycle bits)
- Frequency and resolution are linked — higher resolution = lower max frequency

### Implementation: Passive Buzzer (frequency control)

```c
#include "driver/ledc.h"

#define BUZZER_PIN     GPIO_NUM_19
#define BUZZER_CHANNEL LEDC_CHANNEL_0
#define BUZZER_TIMER   LEDC_TIMER_0

void buzzer_init(void) {
    ledc_timer_config_t timer_cfg = {
        .speed_mode      = LEDC_LOW_SPEED_MODE,
        .timer_num       = BUZZER_TIMER,
        .duty_resolution = LEDC_TIMER_10_BIT,  // 0–1023
        .freq_hz         = 1000,                // 1kHz tone
        .clk_cfg         = LEDC_AUTO_CLK,
    };
    ledc_timer_config(&timer_cfg);

    ledc_channel_config_t ch_cfg = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel    = BUZZER_CHANNEL,
        .timer_sel  = BUZZER_TIMER,
        .intr_type  = LEDC_INTR_DISABLE,
        .gpio_num   = BUZZER_PIN,
        .duty       = 512,   // 50% duty cycle
        .hpoint     = 0,
    };
    ledc_channel_config(&ch_cfg);
}

void buzzer_set_freq(uint32_t freq_hz) {
    ledc_set_freq(LEDC_LOW_SPEED_MODE, BUZZER_TIMER, freq_hz);
}

void buzzer_stop(void) {
    ledc_set_duty(LEDC_LOW_SPEED_MODE, BUZZER_CHANNEL, 0);
    ledc_update_duty(LEDC_LOW_SPEED_MODE, BUZZER_CHANNEL);
}
```

### Implementation: LED Fade (duty cycle control)

```c
#define LED_PWM_PIN     GPIO_NUM_4
#define LED_PWM_CHANNEL LEDC_CHANNEL_1
#define LED_PWM_TIMER   LEDC_TIMER_1

void led_pwm_init(void) {
    ledc_timer_config_t timer_cfg = {
        .speed_mode      = LEDC_LOW_SPEED_MODE,
        .timer_num       = LED_PWM_TIMER,
        .duty_resolution = LEDC_TIMER_8_BIT,  // 0–255
        .freq_hz         = 5000,
        .clk_cfg         = LEDC_AUTO_CLK,
    };
    ledc_timer_config(&timer_cfg);

    ledc_channel_config_t ch_cfg = {
        .speed_mode = LEDC_LOW_SPEED_MODE,
        .channel    = LED_PWM_CHANNEL,
        .timer_sel  = LED_PWM_TIMER,
        .gpio_num   = LED_PWM_PIN,
        .duty       = 0,
        .hpoint     = 0,
    };
    ledc_channel_config(&ch_cfg);
}

void led_fade_task(void *arg) {
    int duty = 0;
    int direction = 1;
    while (1) {
        ledc_set_duty(LEDC_LOW_SPEED_MODE, LED_PWM_CHANNEL, duty);
        ledc_update_duty(LEDC_LOW_SPEED_MODE, LED_PWM_CHANNEL);
        duty += direction * 5;
        if (duty >= 255 || duty <= 0) direction = -direction;
        vTaskDelay(pdMS_TO_TICKS(20));
    }
}
```

### Play a simple melody with the buzzer

```c
typedef struct { uint32_t freq; uint32_t duration_ms; } Note;

static const Note melody[] = {
    {262, 300}, {294, 300}, {330, 300}, {349, 300},
    {392, 400}, {0, 200},   {392, 400},
};

void play_melody_task(void *arg) {
    for (int i = 0; i < sizeof(melody)/sizeof(melody[0]); i++) {
        if (melody[i].freq == 0) {
            buzzer_stop();
        } else {
            buzzer_set_freq(melody[i].freq);
        }
        vTaskDelay(pdMS_TO_TICKS(melody[i].duration_ms));
    }
    buzzer_stop();
    vTaskDelete(NULL);
}
```

**Understand before moving on:**
- What is `duty_resolution` and how does it relate to `freq_hz`?
- Why does 50% duty = `512` for 10-bit and `128` for 8-bit?
- What is the difference between `ledc_set_duty` and `ledc_update_duty`?

---

## Task 5 — Combine Everything in app_main (1 hr)

**Goal:** Wire all tasks and peripherals together cleanly.

```c
void app_main(void) {
    // Create event queue
    gpio_event_queue = xQueueCreate(10, sizeof(GpioEvent));

    // Init peripherals
    gpio_led_init();
    gpio_button_pir_init();
    adc_init();
    buzzer_init();
    led_pwm_init();

    // Start blink timer
    blink_timer_start();

    // Start tasks
    xTaskCreate(gpio_task,      "gpio_task",  2048, NULL, 10, NULL);
    xTaskCreate(adc_read_task,  "adc_task",   2048, NULL,  5, NULL);
    xTaskCreate(led_fade_task,  "fade_task",  2048, NULL,  5, NULL);
    xTaskCreate(play_melody_task,"melody",    2048, NULL,  3, NULL);

    ESP_LOGI(TAG, "All tasks started");
}
```

**Check the monitor output:**
- LED blinks at 2Hz
- Button press toggles LED and logs to serial
- PIR motion logs to serial
- Potentiometer value prints every 200ms
- PWM LED fades up and down
- Melody plays on buzzer at startup

---

## Wrap-up Checklist

Before committing:

- [ ] Project builds with zero warnings: `idf.py build`
- [ ] All peripherals working: LED blink, button ISR, PIR ISR, ADC read, PWM fade, buzzer melody
- [ ] ISR uses `IRAM_ATTR` and only posts to queue — no `printf` in ISR
- [ ] All GPIO pins configured via `gpio_config_t` struct — no individual `gpio_pad_select_gpio` calls
- [ ] No magic numbers — all pins and constants defined with `#define` or `constexpr`
- [ ] Add `NOTES.md`: what surprised you about ESP-IDF vs what you expected, any wiring issues

---

## Commit

```bash
git add week1/session01/
git commit -m "week1/session01: GPIO, interrupts, ADC, PWM, hardware timer — ESP-IDF bare-metal"
git push
```

---

## What to Read Tonight (optional, 30 min)

- [ESP32 Technical Reference Manual](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf) — Chapter 4 (IO MUX and GPIO Matrix), Chapter 13 (LEDC)
- Skim the register descriptions — you won't memorize them, but knowing where to look is the skill
