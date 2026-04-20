# Session 0.4 — C++ Embedded Patterns: Templates, Namespaces, Inline, Static Allocation

> Week 0 | Day 4 | Duration: ~8 hours  
> Platform: PC (no hardware required)  
> Tools: GCC (g++), Make or CMake, any editor/IDE  
> Prerequisite: Sessions 0.1–0.3 complete  
> Commit target: end of session

---

## Objective

Learn the C++ features that are safe, useful, and commonly used in embedded firmware — and understand which C++ features to avoid on constrained hardware. The centerpiece deliverable is a statically-allocated ring buffer template class: a data structure you'll use directly in your FreeRTOS and Zephyr work in Weeks 2 and 3.

---

## Setup (~15 min)

- [ ] Create folder: `week0/session04/`
- [ ] Compile flags: `-Wall -Wextra -Werror -g -std=c++14`
- [ ] No dynamic memory today — everything statically allocated

---

## Task 1 — Namespaces (45 min)

**Goal:** Namespaces organize code and prevent name collisions in large codebases. Embedded firmware often uses them to separate HAL, driver, and application layers.

### Exercises

**1.1 Basic namespace**
```cpp
namespace hal {
    void gpio_write(uint8_t pin, uint8_t value);
    uint8_t gpio_read(uint8_t pin);
}

namespace app {
    void gpio_write(uint8_t pin, uint8_t value);  // different implementation
}
```
- Implement both versions
- Call each explicitly: `hal::gpio_write(...)` and `app::gpio_write(...)`
- Observe: no name collision despite identical function signatures

**1.2 Nested namespaces**
```cpp
namespace adi {
    namespace drivers {
        namespace spi {
            void init();
            void transfer(uint8_t *tx, uint8_t *rx, size_t len);
        }
    }
}
```
- Implement stub functions
- In C++17 you can write `namespace adi::drivers::spi {}` — try both forms
- Call with full path: `adi::drivers::spi::init()`

**1.3 `using` declarations — and when NOT to use them**
```cpp
using namespace hal;       // imports everything — avoid in headers
using hal::gpio_write;     // imports one name — acceptable in .cpp files
```
- Demonstrate why `using namespace X` in a header is dangerous (pollutes every file that includes it)
- Rule: never `using namespace` in a header file; use sparingly in `.cpp` files

**1.4 Anonymous namespaces**
```cpp
namespace {
    static uint8_t internal_buffer[64];  // file-local — like static at file scope in C
    void helper_function();
}
```
- In C++, anonymous namespaces replace `static` for file-local linkage
- Implement a file-local helper using an anonymous namespace

---

## Task 2 — `inline` Functions and `constexpr` (1 hr)

**Goal:** Replace macros with type-safe, debuggable alternatives. This is one of the most impactful C-to-C++ improvements for embedded code.

### Exercises

**2.1 `inline` vs macros**

C macro (problematic):
```c
#define MAX(a, b) ((a) > (b) ? (a) : (b))
```

Problem:
```c
int x = 5;
MAX(x++, 3);   // x incremented twice — undefined behavior
```

C++ inline function (safe):
```cpp
template <typename T>
inline T max_val(T a, T b) { return a > b ? a : b; }
```

- Demonstrate the macro bug
- Show the inline function version is safe
- Rewrite your Day 2 bitmask macros as `inline` functions:
```cpp
inline void bit_set(volatile uint32_t &reg, uint8_t n)   { reg |=  (1U << n); }
inline void bit_clear(volatile uint32_t &reg, uint8_t n) { reg &= ~(1U << n); }
inline bool bit_test(uint32_t reg, uint8_t n)            { return (reg & (1U << n)) != 0; }
```

**2.2 `constexpr` for compile-time constants**
```cpp
constexpr uint32_t CPU_FREQ_HZ    = 240'000'000;
constexpr uint32_t UART_BAUD      = 115200;
constexpr uint32_t BAUD_DIVIDER   = CPU_FREQ_HZ / UART_BAUD;
```
- `constexpr` is evaluated at compile time — zero runtime cost
- Replace all your `#define` numeric constants with `constexpr`
- Write a `constexpr` function:
```cpp
constexpr uint32_t ms_to_ticks(uint32_t ms, uint32_t tick_rate_hz) {
    return ms * tick_rate_hz / 1000;
}
constexpr uint32_t TIMEOUT_TICKS = ms_to_ticks(500, 1000);
```

**2.3 `constexpr` array**
```cpp
constexpr uint8_t VALID_PINS[] = {2, 4, 5, 13, 18, 19, 21, 22};
constexpr size_t  VALID_PIN_COUNT = sizeof(VALID_PINS) / sizeof(VALID_PINS[0]);
```
- Write a `constexpr` function `is_valid_pin(uint8_t pin)` that checks membership
- Evaluated at compile time if called with a compile-time constant

---

## Task 3 — Templates: Basics (1.5 hrs)

**Goal:** Templates let you write type-generic code without runtime overhead — the compiler generates a specialized version for each type you use. This is how the C++ standard library containers work, and it's how embedded libraries like ETL (Embedded Template Library) work.

### Exercises

**3.1 Function template**
```cpp
template <typename T>
T clamp(T value, T min_val, T max_val) {
    if (value < min_val) return min_val;
    if (value > max_val) return max_val;
    return value;
}
```
- Implement and test with `int`, `float`, and `uint8_t`
- Observe: one implementation, multiple compiled versions

**3.2 Class template — basic**
```cpp
template <typename T>
class Optional {
public:
    Optional() : has_value_(false) {}
    Optional(T value) : value_(value), has_value_(true) {}

    bool has_value() const { return has_value_; }
    T value() const { return value_; }

private:
    T    value_;
    bool has_value_;
};
```
- Implement and test with `uint8_t` and a struct
- This is a simplified version of `std::optional` — used when a value may or may not exist (e.g. sensor read that may fail)

**3.3 Non-type template parameters**
```cpp
template <typename T, size_t N>
class StaticArray {
public:
    T& operator[](size_t index) { return data_[index]; }
    const T& operator[](size_t index) const { return data_[index]; }
    size_t size() const { return N; }

private:
    T data_[N];
};
```
- `N` is a compile-time size — no heap, no dynamic sizing
- Implement and test: `StaticArray<uint8_t, 64> buf;`
- This is how embedded libraries avoid `std::vector` (which uses heap)

---

## Task 4 — Main Deliverable: Ring Buffer Template Class (2.5 hrs)

**Goal:** A ring buffer (circular buffer) is used constantly in embedded systems — UART receive buffers, FreeRTOS queues, DMA ping-pong buffers. Build one that is: statically allocated, type-generic, and thread-safe-aware.

### Specification

```cpp
template <typename T, size_t Capacity>
class RingBuffer {
public:
    RingBuffer();

    bool push(const T &item);   // returns false if full
    bool pop(T &item);          // returns false if empty
    bool peek(T &item) const;   // read without removing

    bool   is_empty() const;
    bool   is_full() const;
    size_t size() const;
    size_t capacity() const;
    void   clear();

private:
    T      buffer_[Capacity];
    size_t head_;
    size_t tail_;
    size_t count_;
};
```

**Requirements:**
- No dynamic memory — `buffer_` is a fixed-size array on the stack/static storage
- `Capacity` is a compile-time constant (non-type template parameter)
- `push` when full returns `false` — no overwrite
- `pop` when empty returns `false`
- All size arithmetic uses modulo: `head_ = (head_ + 1) % Capacity`

**Test cases to implement:**
```cpp
RingBuffer<uint8_t, 8> uart_rx;
RingBuffer<uint32_t, 16> sensor_data;

// Test 1: basic push/pop
// Test 2: fill to capacity, verify is_full()
// Test 3: pop from empty, verify returns false
// Test 4: push after pop (wrap-around)
// Test 5: peek doesn't remove item
```

**Stretch goal — thread-safety note:**
Add a comment block explaining:
- This ring buffer is not thread-safe as written
- In FreeRTOS, you'd wrap push/pop in a critical section or use a queue primitive
- In a single-producer single-consumer scenario, head/tail can be `volatile` and atomics can avoid locking

---

## Task 5 — Avoiding the Heap: Embedded Allocation Patterns (45 min)

**Goal:** Know the alternatives to `new`/`delete` and why embedded firmware avoids them.

### Exercises

**5.1 Static allocation**
```cpp
// Instead of:
uint8_t *buf = new uint8_t[256];

// Use:
static uint8_t buf[256];  // lives for program lifetime
```
- When to use: buffers that exist for the lifetime of the program

**5.2 Stack allocation**
```cpp
void process_packet(void) {
    uint8_t tmp[64];  // on the stack, freed on return
    // ...
}
```
- When to use: temporary buffers in a function
- Risk: stack overflow if too large — know your stack size

**5.3 Memory pool (fixed-size allocator)**

Implement a simple memory pool:
```cpp
template <typename T, size_t PoolSize>
class MemoryPool {
public:
    T*   allocate();
    void deallocate(T *ptr);
    bool is_full() const;

private:
    T    pool_[PoolSize];
    bool used_[PoolSize];
};
```
- `allocate()` finds the first unused slot, marks it used, returns pointer
- `deallocate()` marks the slot unused
- O(n) but deterministic — no fragmentation, no non-determinism
- Test: allocate all slots, verify `is_full()`, deallocate one, allocate again

---

## Wrap-up Checklist

Before committing:

- [ ] All code compiles with `g++ -Wall -Wextra -Werror -std=c++14` and zero warnings
- [ ] No `new`/`delete` anywhere — all allocations are static or stack-based
- [ ] `RingBuffer` passes all 5 test cases including wrap-around
- [ ] `MemoryPool` allocates and deallocates correctly
- [ ] `constexpr` used for all numeric constants
- [ ] Add `NOTES.md`: which template concepts felt natural, which were confusing, and what you'd look up again

---

## Commit

```bash
git add week0/session04/
git commit -m "week0/session04: namespaces, constexpr, templates, ring buffer, memory pool"
git push
```

---

## What to Read Tonight (optional, 30 min)

- [Embedded Template Library (ETL)](https://www.etlcpp.com/) — browse the library to see production versions of what you built today (ring buffer, memory pool, optional)
- Tomorrow you wire C and C++ together with `extern "C"` — review your Day 1 C state machine code, you'll be wrapping it
