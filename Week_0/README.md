# Week 0 — C Refresh + C++ Foundations

> Platform: PC (no hardware required)  
> Sessions: 5 weekdays × 8 hrs + optional weekend reading (2 hrs/day)

---

## Goals

- Sharpen embedded-specific C patterns before touching hardware
- Build enough C++ to read, write, and reason about embedded C++ codebases
- Establish GitHub commit habits before Week 1

---

## Sessions

### Session 0.1 — C Refresh: Pointers, Memory, Structs, Function Pointers
**Focus:** Core C patterns you'll use constantly in firmware

Topics:
- Pointer arithmetic, double pointers, pointer-to-function
- Stack vs heap; `malloc`/`free`; when NOT to use heap in embedded
- Structs, typedef, struct padding
- Function pointers and callbacks

**Deliverable:**
- Implement a singly linked list in C (insert, delete, traverse)
- Implement a simple state machine using function pointers

---

### Session 0.2 — C Embedded Patterns: Bit Manipulation, volatile, Register Masking
**Focus:** The low-level C patterns that separate firmware from application code

Topics:
- Fixed-width types: `uint8_t`, `uint32_t`, etc. (`<stdint.h>`)
- Bit manipulation: set, clear, toggle, test a bit
- Bitmasks and bitfields
- `volatile` keyword — why it matters for hardware registers and ISRs
- Register masking patterns: read-modify-write

**Deliverable:**
- Implement a GPIO register emulator using bitfields and masks
- All register operations done via macros or inline functions (no magic numbers)

---

### Session 0.3 — C++ Basics: Classes, RAII, References
**Focus:** The C++ features most relevant to embedded work

Topics:
- Classes, constructors, destructors
- RAII — resource acquisition is initialization; why it matters in embedded
- References vs pointers
- `const` correctness
- `new`/`delete` — and why embedded code often avoids them

**Deliverable:**
- Port your Session 0.1 state machine to a C++ class
- Manage any resources (buffers, handles) using RAII

---

### Session 0.4 — C++ Embedded Patterns: Templates, Namespaces, Inline
**Focus:** C++ features that are safe and useful in constrained environments

Topics:
- Namespaces
- Function and class templates (basics)
- `inline` functions vs macros
- Avoiding dynamic memory: stack allocation, static allocation patterns
- `constexpr` for compile-time constants

**Deliverable:**
- Write a ring buffer (circular buffer) as a C++ template class
- Fixed-size, statically allocated — no heap

---

### Session 0.5 — C + C++ Interop: extern "C"
**Focus:** How C and C++ coexist in real embedded projects

Topics:
- Why C and C++ aren't directly ABI-compatible
- `extern "C"` — what it does and when to use it
- Calling C libraries from C++
- Wrapping C APIs in C++ classes
- Header guards vs `#pragma once`

**Deliverable:**
- Wrap your Session 0.1 C state machine with a C++ interface
- C implementation stays in `.c` files; C++ wrapper in `.hpp`/`.cpp`
- Compiles cleanly with both `gcc` and `g++`

---

## Weekend Reading

| Day | Topic |
|-----|-------|
| Sat | C++ Core Guidelines (embedded subset) — focus on resource management and avoiding exceptions |
| Sun | Google C++ Style Guide — skim for patterns used in large C++ codebases; review your Week 0 code |

---

## Ground Rules

- All code goes on GitHub — commit at end of every session
- No external libraries — standard C/C++ only
- Compile with warnings enabled: `-Wall -Wextra -Werror`
- Write a brief commit message describing what you built and what you learned
