# Session 0.3 — C++ Basics: Classes, RAII, References

> Week 0 | Day 3 | Duration: ~8 hours  
> Platform: PC (no hardware required)  
> Tools: GCC (g++), Make or CMake, any editor/IDE  
> Prerequisite: Sessions 0.1 and 0.2 complete  
> Commit target: end of session

---

## Objective

Build enough C++ to read and write embedded C++ code confidently. The goal is not to master all of C++ — it's to understand the subset that appears in real firmware: classes, RAII, `const` correctness, and how C++ differs from C in ways that matter for embedded work. By the end of this session you will have ported your Day 1 state machine to a C++ class and understand why RAII matters in resource-constrained systems.

---

## Setup (~15 min)

- [ ] Create folder: `week0/session03/`
- [ ] Switch compiler: `g++` instead of `gcc`
- [ ] Compile flags: `-Wall -Wextra -Werror -g -std=c++14`
- [ ] C++14 is the most common standard in embedded projects (Zephyr, ESP-IDF C++ support)

---

## Task 1 — C vs C++: Key Differences (45 min)

**Goal:** Know where C++ diverges from C and why it matters in embedded context.

### Exercises

**1.1 `bool` type**
- C++ has native `bool`; C uses `_Bool` or `<stdbool.h>`
- Write a function `bool is_bit_set(uint32_t reg, uint8_t bit)` — use the C++ `bool` return type
- Verify it works with your bitmask macros from Day 2

**1.2 Default function arguments**
```cpp
void uart_init(uint32_t baud = 115200, uint8_t parity = 0);
```
- Rewrite your Day 2 `uart_init` with default arguments
- Call it with no args, one arg, and both args — observe behavior

**1.3 Function overloading**
```cpp
void print(int value);
void print(float value);
void print(const char *str);
```
- Implement all three — same name, different parameter types
- C cannot do this; C++ resolves at compile time via name mangling

**1.4 References vs pointers**
```cpp
void increment(int &value);   // C++ reference
void increment(int *value);   // C pointer
```
- Implement both versions
- Key difference: references cannot be null, cannot be reseated
- When to use which: references for "must exist" parameters; pointers for optional or array parameters

---

## Task 2 — Classes and Objects (1.5 hrs)

**Goal:** Understand class syntax, access control, and how objects differ from C structs.

### Exercises

**2.1 Basic class**

Convert your Day 2 `GPIO_Regs` struct into a C++ class:

```cpp
class GpioEmulator {
public:
    GpioEmulator();                              // constructor
    void set_mode(uint8_t pin, uint8_t mode);
    void write(uint8_t pin, uint8_t value);
    uint8_t read(uint8_t pin) const;
    void set_pull(uint8_t pin, uint8_t pull);
    void dump() const;

private:
    uint32_t mode_;
    uint32_t out_;
    uint32_t in_;
    uint32_t pull_;
};
```

- Implement all methods
- Note the `const` after `read()` and `dump()` — these methods don't modify the object
- Private members use trailing underscore (`mode_`) — common embedded/Google style

**2.2 Constructor and destructor**
```cpp
class Buffer {
public:
    Buffer(size_t size);
    ~Buffer();
    void write(uint8_t byte);
    uint8_t read();

private:
    uint8_t *data_;
    size_t   size_;
    size_t   head_;
    size_t   tail_;
};
```
- Constructor allocates `data_` with `new`
- Destructor frees it with `delete[]`
- Implement `write()` and `read()` as a simple circular buffer
- This is a preview of the ring buffer you'll make with templates tomorrow

**2.3 `const` correctness**
- Add a `const GpioEmulator` object
- Try to call `write()` on it — observe the compiler error
- Add `const` to the appropriate methods to fix it
- Rule: any method that doesn't modify the object should be marked `const`

---

## Task 3 — RAII (Resource Acquisition Is Initialization) (1.5 hrs)

**Goal:** RAII is the single most important C++ concept for embedded work. It guarantees resources are released when an object goes out of scope — even if an error occurs. No `goto cleanup`, no forgotten `free()`.

### Background

In C, resource management looks like this:
```c
int do_work(void) {
    uint8_t *buf = malloc(256);
    if (!buf) return -1;

    FILE *f = fopen("log.txt", "w");
    if (!f) {
        free(buf);       // easy to forget
        return -1;
    }

    // ... do work ...

    fclose(f);
    free(buf);
    return 0;
}
```

Every early return requires manually releasing everything acquired so far. In C++ with RAII, the destructor handles it automatically.

### Exercises

**3.1 RAII wrapper for a buffer**

```cpp
class ScopedBuffer {
public:
    explicit ScopedBuffer(size_t size) : data_(new uint8_t[size]), size_(size) {}
    ~ScopedBuffer() { delete[] data_; }

    uint8_t* data() { return data_; }
    size_t   size() const { return size_; }

    // Disable copy (important — implement these)
    ScopedBuffer(const ScopedBuffer&) = delete;
    ScopedBuffer& operator=(const ScopedBuffer&) = delete;

private:
    uint8_t *data_;
    size_t   size_;
};
```

- Implement and test: create a `ScopedBuffer`, use it, let it go out of scope
- Add a print statement in the destructor to confirm it's called
- Try to copy it — confirm the compiler rejects it

**3.2 RAII for a simulated peripheral lock**

In embedded systems, you often need to disable interrupts or acquire a mutex before accessing a peripheral. RAII is perfect for this:

```cpp
class CriticalSection {
public:
    CriticalSection()  { enter_critical(); }
    ~CriticalSection() { exit_critical(); }

    CriticalSection(const CriticalSection&) = delete;
    CriticalSection& operator=(const CriticalSection&) = delete;

private:
    void enter_critical() { printf("[IRQ disabled]\n"); }
    void exit_critical()  { printf("[IRQ enabled]\n");  }
};
```

- Use it like this:
```cpp
void write_register(volatile uint32_t *reg, uint32_t value) {
    CriticalSection cs;   // interrupts disabled here
    *reg = value;
}                          // destructor fires here — interrupts re-enabled
```
- Verify: even if an exception is thrown (or early return), the destructor fires
- This pattern is used directly in FreeRTOS and Zephyr C++ wrappers

**3.3 The Rule of Three**

When a class manages a resource, it needs:
1. Destructor — to release the resource
2. Copy constructor — to handle copying correctly (or delete it)
3. Copy assignment operator — same

Implement all three for `ScopedBuffer`:
```cpp
// Copy constructor
ScopedBuffer(const ScopedBuffer& other);

// Copy assignment
ScopedBuffer& operator=(const ScopedBuffer& other);
```
- Deep copy: allocate new memory and copy the data
- Then test: copy a buffer, modify one, verify the other is unchanged

---

## Task 4 — Port State Machine to C++ (2 hrs)

**Goal:** Take your Day 1 C state machine and rewrite it as a proper C++ class. This is your main deliverable for today.

### Specification

```cpp
enum class State  { Red, Green, Yellow };
enum class Event  { Timer, Emergency };

class TrafficLight {
public:
    TrafficLight();
    void process(Event event);
    State state() const;
    void print_state() const;

private:
    State current_;

    State handle_red(Event event);
    State handle_green(Event event);
    State handle_yellow(Event event);

    using StateHandler = State (TrafficLight::*)(Event);
    StateHandler handlers_[3];
};
```

Key points:
- `enum class` instead of `enum` — scoped, no implicit int conversion
- Member function pointers (`StateHandler`) for the dispatch table
- `process()` looks up the handler for `current_` and calls it
- No `switch` on state for dispatch — use the handler table

**Implementation requirements:**
- `TrafficLight::TrafficLight()` — initializes state to `Red`, populates `handlers_`
- Each `handle_*` method returns the next state based on the event
- `process()` calls the appropriate handler via the table
- `print_state()` prints current state as a string

**Test in `main()`:**
```cpp
TrafficLight light;
Event events[] = { Event::Timer, Event::Timer, Event::Timer,
                   Event::Emergency, Event::Timer };

for (Event e : events) {
    light.process(e);
    light.print_state();
}
```

---

## Task 5 — `explicit`, `nullptr`, and Embedded-Relevant C++ Features (45 min)

**Goal:** Small but important C++ features that appear in real embedded codebases.

### Exercises

**5.1 `explicit` constructors**
```cpp
class Pin {
public:
    explicit Pin(uint8_t number) : number_(number) {}
private:
    uint8_t number_;
};
```
- Without `explicit`: `Pin p = 5;` would compile (implicit conversion)
- With `explicit`: `Pin p = 5;` fails; must write `Pin p(5);`
- Why this matters: prevents accidental implicit conversions in hardware APIs

**5.2 `nullptr` vs `NULL`**
- In C++, use `nullptr` — it's type-safe
- `NULL` is typically `0` or `(void*)0` — causes ambiguity with overloaded functions
- Write an overloaded function where `NULL` is ambiguous but `nullptr` is not

**5.3 Range-based for loop**
```cpp
uint8_t pins[] = {2, 5, 13, 21};
for (uint8_t pin : pins) {
    gpio.set_mode(pin, OUTPUT);
}
```
- Rewrite your `gpio_dump()` using a range-based for loop over a pins array

---

## Wrap-up Checklist

Before committing:

- [ ] All code compiles with `g++ -Wall -Wextra -Werror -std=c++14` and zero warnings
- [ ] `GpioEmulator` class works correctly — all methods tested
- [ ] `ScopedBuffer` and `CriticalSection` RAII classes work — destructor fires verified
- [ ] `TrafficLight` state machine works — all transitions correct, dispatch via member function pointer table
- [ ] Add `NOTES.md`: what feels natural from C, what feels new, what RAII clicked for you

---

## Commit

```bash
git add week0/session03/
git commit -m "week0/session03: C++ classes, RAII, references, state machine port"
git push
```

---

## What to Read Tonight (optional, 30 min)

- [C++ Core Guidelines: Resource Management](https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#S-resource)
- Focus on R.1 (RAII), R.10 (avoid malloc), R.11 (avoid new/delete directly)
- These guidelines are what ADI and most embedded shops follow
