# Session 1.2 — UART: Serial Communication and Command Parser

> Week 1 | Day 2 | Duration: ~8 hours  
> Platform: ESP32 via ESP-IDF  
> Hardware: ESP32, USB cable (built-in UART0 via USB-to-serial chip)  
> Tools: `idf.py monitor`, or `minicom`/`PuTTY` for raw UART  
> Prerequisite: Session 1.1 complete and working  
> Commit target: end of session

---

## Objective

Implement UART communication from scratch using ESP-IDF's UART driver — not `printf`. Understand TX/RX buffering, interrupt-driven receive, and build a command parser that accepts text commands over serial and responds with actions. UART is your primary debug tool and comms interface for the rest of the training.

---

## Setup (~15 min)

```bash
cd ~/embedded-training/week1
idf.py create-project session02
cd session02
idf.py set-target esp32
```

Copy your GPIO and PWM code from session01 — you'll reuse the LED and buzzer for command responses.

---

## Background: ESP32 UART Hardware (read first, 20 min)

ESP32 has 3 UART controllers:
- **UART0** — connected to USB-to-serial chip on most devkits. Used by `idf.py monitor` and `ESP_LOGI`. Do not reconfigure this.
- **UART1** — general purpose. Default pins: TX=GPIO10, RX=GPIO9 (often conflicts with flash — reassign).
- **UART2** — general purpose. Default pins: TX=GPIO17, RX=GPIO16. Safe to use.

For this session you have two options:
- **Option A:** Use UART0 directly for your command parser (same port as monitor — simpler wiring, slightly trickier to separate log output from command I/O)
- **Option B:** Use UART2 on GPIO16/17 with a USB-to-serial adapter or a second ESP32

**Recommended: Option A with UART0** — simplest setup, teaches the real driver API, and `idf.py monitor` can still be used with `Ctrl+]` to exit and switch to `minicom`.

---

## Task 1 — UART Driver Init (45 min)

**Goal:** Configure UART using the driver API. Understand every parameter.

### Implementation

```c
#include "driver/uart.h"
#include "driver/gpio.h"
#include "esp_log.h"

#define UART_PORT       UART_NUM_0
#define UART_BAUD       115200
#define UART_TX_PIN     GPIO_NUM_1    // UART0 default TX
#define UART_RX_PIN     GPIO_NUM_3    // UART0 default RX
#define UART_BUF_SIZE   1024

static const char *TAG = "uart";

void uart_init(void) {
    uart_config_t cfg = {
        .baud_rate           = UART_BAUD,
        .data_bits           = UART_DATA_8_BITS,
        .parity              = UART_PARITY_DISABLE,
        .stop_bits           = UART_STOP_BITS_1,
        .flow_ctrl           = UART_HW_FLOWCTRL_DISABLE,
        .source_clk          = UART_SCLK_DEFAULT,
    };

    uart_driver_install(UART_PORT, UART_BUF_SIZE, UART_BUF_SIZE, 0, NULL, 0);
    uart_param_config(UART_PORT, &cfg);
    uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN,
                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);
}
```

**Understand every field:**
- `data_bits` — almost always 8 for modern devices
- `parity` — error detection bit; disabled means raw 8N1 (8 data, no parity, 1 stop)
- `stop_bits` — 1 is standard; 2 is legacy
- `flow_ctrl` — hardware flow control (RTS/CTS); disabled for simple setups
- `UART_BUF_SIZE` in `uart_driver_install` — size of internal RX and TX ring buffers in bytes

---

## Task 2 — TX: Sending Data (30 min)

**Goal:** Send data over UART using the driver — not `printf`.

### Implementation

```c
void uart_send_str(const char *str) {
    uart_write_bytes(UART_PORT, str, strlen(str));
}

void uart_send_bytes(const uint8_t *data, size_t len) {
    uart_write_bytes(UART_PORT, (const char *)data, len);
}

void uart_send_formatted(const char *fmt, ...) {
    char buf[256];
    va_list args;
    va_start(args, fmt);
    vsnprintf(buf, sizeof(buf), fmt, args);
    va_end(args);
    uart_write_bytes(UART_PORT, buf, strlen(buf));
}
```

**Test:**
```c
uart_send_str("ESP32 UART ready\r\n");
uart_send_formatted("Uptime: %lu ms\r\n", esp_timer_get_time() / 1000);
```

**Understand:**
- Why `\r\n` instead of just `\n`? (terminal emulators expect CRLF)
- What does `uart_write_bytes` return? Check for errors.
- What happens if you write more than `UART_BUF_SIZE`? It blocks until the buffer drains.

---

## Task 3 — RX: Receiving Data — Polling vs Event-Driven (1.5 hrs)

**Goal:** Implement two receive approaches and understand the tradeoff.

### Approach A: Polling (simple, not ideal for production)

```c
void uart_rx_poll_task(void *arg) {
    uint8_t buf[128];
    while (1) {
        int len = uart_read_bytes(UART_PORT, buf, sizeof(buf) - 1,
                                  pdMS_TO_TICKS(100));
        if (len > 0) {
            buf[len] = '\0';
            ESP_LOGI(TAG, "RX (%d bytes): %s", len, buf);
        }
    }
}
```

Limitation: blocks for up to 100ms per call. Misses data if the buffer fills between polls.

### Approach B: Event-driven with UART event queue (production approach)

```c
#define UART_EVENT_QUEUE_SIZE 10

static QueueHandle_t uart_event_queue;

void uart_init_event_driven(void) {
    uart_config_t cfg = { /* same as before */ };
    uart_driver_install(UART_PORT, UART_BUF_SIZE, UART_BUF_SIZE,
                        UART_EVENT_QUEUE_SIZE, &uart_event_queue, 0);
    uart_param_config(UART_PORT, &cfg);
    uart_set_pin(UART_PORT, UART_TX_PIN, UART_RX_PIN,
                 UART_PIN_NO_CHANGE, UART_PIN_NO_CHANGE);

    // Set pattern detection: newline triggers an event
    uart_enable_pattern_det_baud_intr(UART_PORT, '\n', 1, 9, 0, 0);
    uart_pattern_queue_reset(UART_PORT, 20);
}

void uart_event_task(void *arg) {
    uart_event_t event;
    uint8_t buf[256];

    while (1) {
        if (xQueueReceive(uart_event_queue, &event, portMAX_DELAY)) {
            switch (event.type) {
                case UART_DATA:
                    uart_read_bytes(UART_PORT, buf, event.size, portMAX_DELAY);
                    buf[event.size] = '\0';
                    ESP_LOGI(TAG, "Data: %s", buf);
                    break;

                case UART_PATTERN_DET: {
                    int pos = uart_pattern_pop_pos(UART_PORT);
                    if (pos != -1) {
                        uart_read_bytes(UART_PORT, buf, pos + 1, pdMS_TO_TICKS(100));
                        buf[pos] = '\0';  // strip newline
                        // pass to command parser
                    }
                    break;
                }

                case UART_FIFO_OVF:
                    ESP_LOGW(TAG, "FIFO overflow — flushing");
                    uart_flush_input(UART_PORT);
                    xQueueReset(uart_event_queue);
                    break;

                case UART_BUFFER_FULL:
                    ESP_LOGW(TAG, "Buffer full — flushing");
                    uart_flush_input(UART_PORT);
                    xQueueReset(uart_event_queue);
                    break;

                default:
                    break;
            }
        }
    }
}
```

**Understand:**
- What is `uart_enable_pattern_det_baud_intr`? What does it detect?
- What is `UART_FIFO_OVF` and when does it happen?
- Why reset the queue on overflow rather than just flushing?

---

## Task 4 — Line Buffer and Command Parser (2 hrs)

**Goal:** Build a robust command parser that reads line-by-line, parses command + arguments, and dispatches to handlers. This is a direct analog of your Day 1 function pointer dispatch table.

### Line buffer

```c
typedef struct {
    char   buf[128];
    size_t len;
} LineBuffer;

// Returns true when a complete line is received (terminated by \n or \r\n)
bool linebuf_push(LineBuffer *lb, uint8_t byte) {
    if (byte == '\r') return false;  // ignore CR
    if (byte == '\n') {
        lb->buf[lb->len] = '\0';
        return true;                 // line complete
    }
    if (lb->len < sizeof(lb->buf) - 1) {
        lb->buf[lb->len++] = byte;
    }
    return false;
}

void linebuf_reset(LineBuffer *lb) {
    lb->len = 0;
    lb->buf[0] = '\0';
}
```

### Command table

```c
typedef void (*CommandHandler)(int argc, char **argv);

typedef struct {
    const char     *name;
    CommandHandler  handler;
    const char     *help;
} Command;
```

### Commands to implement

| Command | Arguments | Action |
|---------|-----------|--------|
| `help` | none | Print all commands and descriptions |
| `led on` | none | Turn LED on |
| `led off` | none | Turn LED off |
| `led blink <ms>` | period in ms | Blink LED at given period |
| `buzz <freq>` | frequency in Hz | Play tone on buzzer |
| `buzz off` | none | Stop buzzer |
| `adc read` | none | Read and print ADC value |
| `uptime` | none | Print system uptime in ms |
| `reboot` | none | Restart ESP32 via `esp_restart()` |

### Parser implementation

```c
#define MAX_ARGS 4

static int parse_command(char *line, char **argv) {
    int argc = 0;
    char *tok = strtok(line, " \t");
    while (tok && argc < MAX_ARGS) {
        argv[argc++] = tok;
        tok = strtok(NULL, " \t");
    }
    return argc;
}

static void dispatch_command(char *line) {
    if (strlen(line) == 0) return;

    char *argv[MAX_ARGS];
    int   argc = parse_command(line, argv);
    if (argc == 0) return;

    for (int i = 0; i < NUM_COMMANDS; i++) {
        if (strcmp(argv[0], commands[i].name) == 0) {
            commands[i].handler(argc, argv);
            return;
        }
    }

    uart_send_formatted("Unknown command: %s (type 'help')\r\n", argv[0]);
}
```

### Echo and prompt

```c
// Print prompt after each command
uart_send_str("\r\n> ");

// Echo characters as they're typed (makes it feel like a real terminal)
static void IRAM_ATTR uart_rx_isr(void *arg) {
    // handled via event queue — echo in event task
}
```

---

## Task 5 — Echo and Polish (1 hr)

**Goal:** Make the terminal feel professional — proper echo, backspace support, prompt.

### Echo task

In your UART event task, when `UART_DATA` events arrive (individual bytes before newline):
```c
case UART_DATA: {
    uint8_t ch;
    uart_read_bytes(UART_PORT, &ch, 1, portMAX_DELAY);

    if (ch == '\b' || ch == 127) {   // backspace
        if (lb.len > 0) {
            lb.len--;
            uart_send_str("\b \b");  // erase character from terminal
        }
    } else {
        uart_write_bytes(UART_PORT, (char *)&ch, 1);  // echo
        if (linebuf_push(&lb, ch)) {
            uart_send_str("\r\n");
            dispatch_command(lb.buf);
            linebuf_reset(&lb);
            uart_send_str("> ");
        }
    }
    break;
}
```

### Test the full terminal

Connect via `minicom -b 115200 -D /dev/ttyUSB0` (or PuTTY on Windows):
- Type `help` → see all commands
- Type `led on` → LED turns on
- Type `led blink 200` → LED blinks at 200ms
- Type `buzz 440` → buzzer plays 440Hz (A4)
- Type `adc read` → prints raw and voltage
- Type `reboot` → ESP32 restarts
- Backspace works correctly
- Unknown commands print error message

---

## Wrap-up Checklist

Before committing:

- [ ] UART driver initialized with correct config (not just `printf`)
- [ ] Event-driven RX using `uart_event_queue` — not polling loop
- [ ] All 9 commands implemented and working
- [ ] Backspace and echo work correctly
- [ ] Overflow and buffer-full events handled gracefully
- [ ] Command dispatch uses function pointer table — no `if/else if` chain
- [ ] Add `NOTES.md`: what UART edge cases you hit, how the event queue differs from polling

---

## Commit

```bash
git add week1/session02/
git commit -m "week1/session02: UART driver, event-driven RX, command parser with function pointer dispatch"
git push
```

---

## What to Read Tonight (optional, 30 min)

- [ESP32 TRM](https://www.espressif.com/sites/default/files/documentation/esp32_technical_reference_manual_en.pdf) — Chapter 11 (UART Controller): read the FIFO, baud rate, and interrupt sections
- Understand how the baud rate divider is calculated from the APB clock — you'll see this pattern again for SPI and I2C clock configuration
