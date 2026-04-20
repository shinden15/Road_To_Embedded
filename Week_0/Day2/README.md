# Session 0.2 — C Embedded Patterns: Bit Manipulation, volatile, Register Masking

> Week 0 | Day 2 | Duration: ~8 hours  
> Platform: PC (no hardware required)  
> Tools: GCC, GDB, Make or CMake, any editor/IDE  
> Prerequisite: Session 0.1 complete  
> Commit target: end of session

---

## Objective

Learn the low-level C patterns used to interact with hardware registers. Every peripheral you'll touch on ESP32, RPi, or any MCU is controlled by reading and writing memory-mapped registers using exactly these techniques. By the end of this session you should be able to set, clear, toggle, and test individual bits without thinking about it.

---

## Setup (~15 min)

- [ ] Create folder: `week0/session02/`
- [ ] Copy your `Makefile` or `CMakeLists.txt` from session01
- [ ] Same compile flags: `-Wall -Wextra -Werror -g`
- [ ] Include `<stdint.h>` in all files — use fixed-width types exclusively (`uint8_t`, `uint16_t`, `uint32_t`)

---

## Background Reading (do this first, 30 min)

Before writing any code, read the following — it will make every exercise click:

- What is a memory-mapped peripheral? A hardware register is just a memory address. Writing to it changes hardware state. Reading from it returns hardware state.
- Why does the compiler need `volatile`? Without it, the compiler may cache a register value in a CPU register and never re-read it from memory — even though the hardware may have changed it.
- What is a read-modify-write? When you change one bit in a register, you must preserve all other bits. The pattern is: read the current value, modify the target bit(s), write back.

---

## Task 1 — Fixed-Width Types (30 min)

**Goal:** Stop using `int` and `long` in embedded code. Sizes are platform-dependent. Fixed-width types are not.

### Exercises

**1.1 Size verification**
```c
#include <stdint.h>
#include <stdio.h>
```
- Print `sizeof` for: `char`, `short`, `int`, `long`, `long long`, `uint8_t`, `uint16_t`, `uint32_t`, `uint64_t`
- Observe which native types match which fixed-width types on your platform

**1.2 Type limits**
```c
#include <stdint.h>
#include <limits.h>
```
- Print `UINT8_MAX`, `UINT16_MAX`, `UINT32_MAX`
- Print `INT8_MIN`, `INT8_MAX`
- Write a function that detects overflow before it happens:
```c
int safe_add_u8(uint8_t a, uint8_t b, uint8_t *result);
```
Returns 0 on success, -1 on overflow. Overflow detection is common in embedded safety-critical code.

---

## Task 2 — Bit Manipulation Fundamentals (1.5 hrs)

**Goal:** Set, clear, toggle, and test individual bits. These four operations cover ~90% of register work.

### Reference

```c
// Set bit N
value |= (1U << N);

// Clear bit N
value &= ~(1U << N);

// Toggle bit N
value ^= (1U << N);

// Test bit N (returns non-zero if set)
value & (1U << N);
```

### Exercises

**2.1 Implement bit operation macros**
```c
#define BIT_SET(reg, n)    ((reg) |=  (1U << (n)))
#define BIT_CLEAR(reg, n)  ((reg) &= ~(1U << (n)))
#define BIT_TOGGLE(reg, n) ((reg) ^=  (1U << (n)))
#define BIT_TEST(reg, n)   ((reg) &   (1U << (n)))
```
- Write a test for each macro on a `uint8_t` and a `uint32_t`
- Print the register value in binary after each operation to verify

**2.2 Print binary helper**

Write a function:
```c
void print_binary(uint32_t value, uint8_t num_bits);
```
- Prints `value` as a binary string, MSB first, `num_bits` wide
- Example: `print_binary(0b10110001, 8)` → `10110001`
- You'll use this constantly to verify bit operations

**2.3 Multi-bit field extraction**

Given a 32-bit register, extract a field of `width` bits starting at `offset`:
```c
uint32_t extract_field(uint32_t reg, uint8_t offset, uint8_t width);
```
- Example: `extract_field(0xABCD1234, 8, 8)` → `0x12`
- This is how you read multi-bit peripheral configuration fields

**2.4 Multi-bit field insertion**

Write the inverse:
```c
uint32_t insert_field(uint32_t reg, uint8_t offset, uint8_t width, uint32_t value);
```
- Inserts `value` into `reg` at `offset` for `width` bits
- Must not corrupt other bits (read-modify-write)

---

## Task 3 — Bitmasks and Bitfields (1.5 hrs)

**Goal:** Two ways to describe register layouts — bitmasks (more common in C firmware) and bitfields (convenient but with caveats).

### Exercises

**3.1 Bitmask approach**

Model a fictional `CTRL` register for a UART peripheral:

```
Bit 7:   UART_EN     (enable)
Bit 6:   UART_TX_EN  (TX enable)
Bit 5:   UART_RX_EN  (RX enable)
Bit 4-3: UART_PARITY (00=none, 01=even, 10=odd)
Bit 2-1: UART_STOP   (00=1bit, 01=1.5bit, 10=2bit)
Bit 0:   UART_LOOP   (loopback mode)
```

Define masks and shifts as `#define` constants:
```c
#define UART_EN_BIT       7
#define UART_EN_MASK      (1U << UART_EN_BIT)
#define UART_PARITY_SHIFT 3
#define UART_PARITY_MASK  (0x3U << UART_PARITY_SHIFT)
// ... and so on
```

Write functions:
```c
uint8_t uart_ctrl_set_enable(uint8_t reg, uint8_t enable);
uint8_t uart_ctrl_set_parity(uint8_t reg, uint8_t parity);
uint8_t uart_ctrl_get_parity(uint8_t reg);
```

**3.2 Bitfield approach**

Model the same register using a C struct bitfield:
```c
typedef struct __attribute__((packed)) {
    uint8_t loop    : 1;
    uint8_t stop    : 2;
    uint8_t parity  : 2;
    uint8_t rx_en   : 1;
    uint8_t tx_en   : 1;
    uint8_t uart_en : 1;
} UartCtrlReg;
```

- Set the same fields as in 3.1 using the struct
- Print the raw byte value — verify it matches your bitmask approach

**3.3 Why bitmasks are preferred in firmware**

Write a comment block explaining:
- Bitfield bit ordering is implementation-defined (compiler-dependent)
- Bitmasks are explicit and portable
- When you might still use bitfields (internal structs, not hardware-mapped)

---

## Task 4 — `volatile` Keyword (1 hr)

**Goal:** Understand what `volatile` does and when you must use it. This is a common interview question.

### Exercises

**4.1 Simulate a hardware register**

```c
volatile uint32_t FAKE_GPIO_IN = 0;
```

- Write a polling loop that waits until bit 3 of `FAKE_GPIO_IN` is set
- In a separate thread or via a signal handler, set that bit after 1 second
- Without `volatile`: the compiler may optimize the loop into an infinite `if(0)` check
- With `volatile`: the compiler re-reads the value every iteration

**4.2 `volatile` in ISR context**

```c
volatile uint8_t flag = 0;

void isr_handler(void) {
    flag = 1;  // set by interrupt
}

int main(void) {
    while (!flag) { /* wait */ }
    // process event
}
```

- Implement this pattern (simulate the ISR with a signal handler: `signal(SIGALRM, isr_handler)`)
- Set the alarm: `alarm(2)` — ISR fires after 2 seconds
- Without `volatile` on `flag`, the compiler may cache it and never see the update

**4.3 `volatile const`**

- What does `volatile const uint32_t *reg` mean?
- Write a short explanation: `const` means you won't write to it; `volatile` means hardware may change it
- Common use: read-only status registers

---

## Task 5 — Register Masking Patterns (1 hr)

**Goal:** Practice the full read-modify-write cycle that you'll use for every peripheral configuration in firmware.

### Exercises

**5.1 Read-modify-write**

Given a simulated 32-bit peripheral control register:
```c
volatile uint32_t PERIPH_CTRL = 0x00000000;
```

Write a function that configures bits 15:12 (a 4-bit clock divider field) to a given value without touching any other bits:
```c
void set_clock_divider(volatile uint32_t *reg, uint8_t divider);
```

**5.2 Atomic-safe pattern**

In real firmware, read-modify-write is a problem if an interrupt fires between the read and the write. Simulate this:
- Disable interrupts (use a mutex or flag in your simulation)
- Do the read-modify-write
- Re-enable interrupts

Pattern:
```c
ENTER_CRITICAL();
reg = (reg & ~MASK) | (value << SHIFT);
EXIT_CRITICAL();
```

**5.3 Register emulator**

Build a small GPIO register emulator:

```c
typedef struct {
    volatile uint32_t MODE;    // 0=input, 1=output per pin
    volatile uint32_t OUT;     // output values
    volatile uint32_t IN;      // input values (read-only)
    volatile uint32_t PULL;    // 0=none, 1=pull-up, 2=pull-down
} GPIO_Regs;

static GPIO_Regs gpio = {0};
```

Implement:
```c
void gpio_set_mode(uint8_t pin, uint8_t mode);
void gpio_write(uint8_t pin, uint8_t value);
uint8_t gpio_read(uint8_t pin);
void gpio_set_pull(uint8_t pin, uint8_t pull);
void gpio_dump(void);  // print all register values in binary
```

- All operations use bitmask patterns only — no magic numbers
- `gpio_dump()` uses your `print_binary()` from Task 2

---

## Task 6 — Putting It Together: UART Register Emulator (1.5 hrs)

**Goal:** Combine everything from today into a realistic register-level peripheral emulator.

### Specification

Implement a UART register emulator with the following registers:

```c
typedef struct {
    volatile uint8_t CTRL;    // control register (from Task 3)
    volatile uint8_t STATUS;  // bit0=TX_READY, bit1=RX_READY, bit2=ERROR
    volatile uint8_t DATA;    // TX/RX data register
    volatile uint8_t BAUD;    // baud rate divider
} UART_Regs;
```

Implement:
```c
void uart_init(UART_Regs *uart, uint8_t baud_divider);
void uart_enable(UART_Regs *uart);
void uart_disable(UART_Regs *uart);
int  uart_send(UART_Regs *uart, uint8_t data);   // returns -1 if not ready
int  uart_recv(UART_Regs *uart, uint8_t *data);  // returns -1 if no data
void uart_dump(UART_Regs *uart);                  // print all registers in binary
```

- All register access through bitmasks and macros — no direct struct field assignment for bit-level ops
- Simulate TX_READY/RX_READY flags with a simple state variable
- `uart_dump()` prints every register in binary with field annotations

---

## Wrap-up Checklist

Before committing:

- [ ] All tasks compile with `-Wall -Wextra -Werror` and zero warnings
- [ ] `print_binary()` used throughout to verify bit operations visually
- [ ] No magic numbers — all bits defined as named constants or macros
- [ ] GPIO and UART emulators work correctly end-to-end
- [ ] Add `NOTES.md`: what was rusty, what surprised you, what you'd do differently

---

## Commit

```bash
git add week0/session02/
git commit -m "week0/session02: bit manipulation, volatile, bitmasks, GPIO and UART register emulators"
git push
```

---

## What to Read Tonight (optional, 30 min)

- Skim the ESP32 Technical Reference Manual — find the GPIO register map (Chapter on IO MUX and GPIO Matrix). You'll implement this for real in Week 1.
- Note: the patterns from today are exactly how ESP-IDF's low-level HAL works internally.
