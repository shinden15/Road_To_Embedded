# Session 0.5 — C + C++ Interop: extern "C", Mixing C and C++

> Week 0 | Day 5 | Duration: ~8 hours  
> Platform: PC (no hardware required)  
> Tools: GCC + G++, Make or CMake, any editor/IDE  
> Prerequisite: Sessions 0.1–0.4 complete  
> Commit target: end of session

---

## Objective

Real embedded projects almost always mix C and C++. Vendor SDKs (ESP-IDF, Zephyr HAL, CMSIS) are written in C. Your application layer may be C++. You need to call C from C++ and sometimes C++ from C cleanly. By the end of this session you'll understand why this is necessary, how `extern "C"` works, and you'll have a complete project that combines a C firmware module with a C++ application layer — the same pattern you'll see in Week 1 onward.

---

## Setup (~15 min)

- [ ] Create folder: `week0/session05/`
- [ ] This session has multiple files — set up a proper `CMakeLists.txt` or `Makefile` with separate compilation units
- [ ] C files compiled with `gcc`; C++ files compiled with `g++`; linked together

**Suggested file structure:**
```
session05/
├── CMakeLists.txt        (or Makefile)
├── c_lib/
│   ├── state_machine.h   (C header)
│   ├── state_machine.c   (C implementation)
│   ├── uart_emulator.h   (C header)
│   └── uart_emulator.c   (C implementation from Day 2)
├── cpp_app/
│   ├── device.hpp        (C++ class wrapping C modules)
│   ├── device.cpp
│   └── ring_buffer.hpp   (your Day 4 template)
└── main.cpp              (C++ entry point)
```

---

## Background: Why `extern "C"` Exists (read before coding, 20 min)

**Name mangling:** When g++ compiles a C++ function, it encodes the parameter types into the symbol name. This is how function overloading works — `void send(int)` and `void send(float)` become different symbols.

```
C:    send        → symbol: send
C++:  send(int)   → symbol: _Z4sendi
C++:  send(float) → symbol: _Z4sendf
```

When you try to call a C function from C++, the linker looks for `_Z4sendi` but the C object file only has `send`. Link error.

`extern "C"` tells the C++ compiler: "don't mangle this name — treat it as a C symbol."

---

## Task 1 — Understanding Name Mangling (45 min)

**Goal:** See the problem firsthand before learning the fix.

### Exercises

**1.1 Observe mangled names**
```bash
# Compile a C++ file with a few functions
g++ -c functions.cpp -o functions.o
nm functions.o         # list symbols
c++filt _Z4sendi       # demangle a symbol
```
- Write 3 overloaded functions in C++
- Run `nm` and `c++filt` on the object file
- Observe the mangled names

**1.2 Break it intentionally**
- Write a C function in `helper.c`
- Try to call it from `main.cpp` WITHOUT `extern "C"`
- Observe the linker error — read it carefully, understand what's missing

**1.3 Fix it with `extern "C"`**
```c
// helper.h
#ifdef __cplusplus
extern "C" {
#endif

void helper_function(uint8_t value);

#ifdef __cplusplus
}
#endif
```
- Add the `extern "C"` guard to `helper.h`
- Recompile — linker error gone
- Understand: `__cplusplus` is defined by the C++ compiler but not the C compiler, so the header works in both contexts

---

## Task 2 — Wrap the Day 1 State Machine in C++ (1.5 hrs)

**Goal:** Take your C state machine from Day 1 and expose it through a clean C++ interface.

### Step 1: C module (keep it pure C)

`c_lib/state_machine.h`:
```c
#ifndef STATE_MACHINE_H
#define STATE_MACHINE_H

#ifdef __cplusplus
extern "C" {
#endif

typedef enum {
    SM_STATE_RED = 0,
    SM_STATE_GREEN,
    SM_STATE_YELLOW,
    SM_STATE_COUNT
} SmState;

typedef enum {
    SM_EVENT_TIMER = 0,
    SM_EVENT_EMERGENCY,
    SM_EVENT_COUNT
} SmEvent;

typedef struct {
    SmState current;
} StateMachine;

void sm_init(StateMachine *sm);
void sm_process(StateMachine *sm, SmEvent event);
const char* sm_state_name(SmState state);

#ifdef __cplusplus
}
#endif

#endif
```

`c_lib/state_machine.c`:
- Implement using your Day 1 function pointer table
- No C++ — pure C, compiled with `gcc`

### Step 2: C++ wrapper

`cpp_app/device.hpp`:
```cpp
#pragma once
#include "state_machine.h"  // C header — safe via extern "C"

namespace app {

enum class LightState  { Red, Green, Yellow };
enum class LightEvent  { Timer, Emergency };

class TrafficLight {
public:
    TrafficLight();

    void process(LightEvent event);
    LightState state() const;
    const char* state_name() const;

private:
    StateMachine sm_;  // C struct — embedded directly, no pointer, no heap

    static SmEvent to_c_event(LightEvent event);
    static LightState to_cpp_state(SmState state);
};

} // namespace app
```

`cpp_app/device.cpp`:
- Implement the wrapper — it translates between C++ enums and C enums, delegates all logic to the C module
- `sm_` is a value member, not a pointer — no heap allocation

### Step 3: Verify in `main.cpp`
```cpp
#include "cpp_app/device.hpp"

int main() {
    app::TrafficLight light;

    light.process(app::LightEvent::Timer);
    light.process(app::LightEvent::Timer);
    light.process(app::LightEvent::Emergency);

    return 0;
}
```

---

## Task 3 — Wrap the Day 2 UART Emulator in C++ (1.5 hrs)

**Goal:** Same pattern — C implementation, C++ interface.

### Step 1: C module

`c_lib/uart_emulator.h` and `c_lib/uart_emulator.c`:
- Port your Day 2 UART emulator as a pure C module
- Add `extern "C"` guards to the header

### Step 2: C++ wrapper

```cpp
namespace hal {

class Uart {
public:
    explicit Uart(uint8_t baud_divider);
    ~Uart() = default;

    bool send(uint8_t data);
    bool recv(uint8_t &data);
    void dump() const;

    // Non-copyable
    Uart(const Uart&) = delete;
    Uart& operator=(const Uart&) = delete;

private:
    UART_Regs regs_;
};

} // namespace hal
```

- `UART_Regs` is the C struct — embedded as a value member
- Constructor calls `uart_init()` from the C module
- Methods call the corresponding C functions

### Step 3: Use RingBuffer with UART

Connect your Day 4 `RingBuffer` to the UART wrapper:
```cpp
hal::Uart uart(4);
RingBuffer<uint8_t, 64> rx_buffer;

// Simulate receiving bytes and buffering them
for (uint8_t byte : {0x41, 0x42, 0x43}) {
    rx_buffer.push(byte);
}

uint8_t b;
while (rx_buffer.pop(b)) {
    uart.send(b);
}
```

---

## Task 4 — Calling C++ from C (1 hr)

**Goal:** Less common but it comes up — a C ISR or callback needs to call into a C++ object.

### The problem

C doesn't know about C++ objects. You can't pass a C++ class instance directly to a C callback.

### The solution: C-linkage wrapper functions

```cpp
// In C++ file — provides C-callable interface to a C++ object
extern "C" void uart_rx_isr_handler(void) {
    // Access the C++ object through a static pointer or global
    if (g_uart_instance) {
        g_uart_instance->on_rx_interrupt();
    }
}
```

### Exercises

**4.1 Static instance pattern**
```cpp
// device.cpp
static app::TrafficLight *g_instance = nullptr;

extern "C" void traffic_light_set_instance(void *instance) {
    g_instance = static_cast<app::TrafficLight*>(instance);
}

extern "C" void traffic_light_process_timer(void) {
    if (g_instance) g_instance->process(app::LightEvent::Timer);
}
```
- Implement in `.cpp`, call from a `.c` file
- This is exactly how FreeRTOS task functions work with C++ objects

**4.2 Callback registration pattern**
```cpp
typedef void (*EventCallback)(uint8_t event_id);

class EventDispatcher {
public:
    void register_callback(EventCallback cb) { callback_ = cb; }
    void dispatch(uint8_t event) {
        if (callback_) callback_(event);
    }
private:
    EventCallback callback_ = nullptr;
};
```
- A C function registered as a callback into a C++ object
- Test: register a C function, trigger dispatch, verify it's called

---

## Task 5 — Build System: Compiling Mixed C and C++ (1 hr)

**Goal:** A working `CMakeLists.txt` (or `Makefile`) that compiles C and C++ separately and links them together. This is the build structure you'll use in ESP-IDF and Zephyr.

### CMakeLists.txt (recommended)

```cmake
cmake_minimum_required(VERSION 3.14)
project(week0_session05 C CXX)

set(CMAKE_C_STANDARD 11)
set(CMAKE_CXX_STANDARD 14)

add_compile_options(-Wall -Wextra -Werror)

# C library
add_library(c_lib STATIC
    c_lib/state_machine.c
    c_lib/uart_emulator.c
)

# C++ application
add_executable(app
    main.cpp
    cpp_app/device.cpp
)

target_include_directories(app PRIVATE .)
target_link_libraries(app PRIVATE c_lib)
```

**Verify:**
```bash
mkdir build && cd build
cmake ..
make
./app
```

### Exercises

- Add a deliberate `extern "C"` error (remove it from one header) — observe the exact linker error
- Fix it — observe it builds cleanly
- Run `nm build/app | grep sm_` — verify C symbols are unmangled in the final binary

---

## Wrap-up Checklist

Before committing:

- [ ] Project builds cleanly: `cmake .. && make` with zero warnings and zero errors
- [ ] C modules (`state_machine.c`, `uart_emulator.c`) are pure C — no C++ constructs
- [ ] C headers have `extern "C"` guards — work correctly from both C and C++ files
- [ ] C++ wrappers use value members (no heap) for C structs
- [ ] `RingBuffer` integrated with `Uart` wrapper
- [ ] C-from-C++ and C++-from-C both demonstrated and working
- [ ] Add `NOTES.md`: what confused you about the build system, what clicked about `extern "C"`

---

## Commit

```bash
git add week0/session05/
git commit -m "week0/session05: extern C, C/C++ interop, mixed build system, C wrappers"
git push
```

---

## Week 0 Complete — Review Before Week 1

You now have:
- Solid C embedded patterns: pointers, memory, bit manipulation, register emulation
- Enough C++ for real firmware: classes, RAII, templates, namespaces
- C/C++ interop: `extern "C"`, mixed builds — the pattern used in every vendor SDK

**Check before moving to Week 1:**
- [ ] All 5 sessions committed to GitHub
- [ ] Each session folder has a `NOTES.md` or comments documenting what you learned
- [ ] You can explain `volatile`, RAII, and `extern "C"` out loud without notes
- [ ] Hardware ordered and tracking (logic analyzer, ST-Link, sensors)

Week 1 starts on the ESP32 — real hardware, real registers, ESP-IDF only.
