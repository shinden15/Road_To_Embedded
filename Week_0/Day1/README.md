# Session 0.1 — C Refresh: Pointers, Memory, Structs, Function Pointers

> Week 0 | Day 1 | Duration: ~8 hours  
> Platform: PC (Linux/Windows/Mac — no hardware required)  
> Tools: GCC, GDB, Make or CMake, any editor/IDE  
> Commit target: end of session

---

## Objective

Refresh core C patterns that appear constantly in embedded firmware. By the end of this session you should be able to write and reason about pointer-heavy C code, manage memory manually, and use function pointers for callbacks and dispatch tables — all without looking things up.

---

## Setup (do first, ~15 min)

- [ ] Create GitHub repo if not yet done: `embedded-training`
- [ ] Create folder: `week0/session01/`
- [ ] Verify GCC is installed: `gcc --version`
- [ ] Verify GDB is installed: `gdb --version`
- [ ] Create a `Makefile` or `CMakeLists.txt` for the session
- [ ] Compile with: `-Wall -Wextra -Werror -g`

---

## Task 1 — Pointers and Pointer Arithmetic (1 hr)

**Goal:** Be comfortable with pointers at the level firmware code expects.

### Exercises

**1.1 Basic pointer operations**
```c
int x = 42;
int *p = &x;
```
- Print the address of `x` using `%p`
- Modify `x` through `p`
- Print both `p` and `*p` — understand the difference

**1.2 Pointer arithmetic**
```c
int arr[5] = {10, 20, 30, 40, 50};
int *p = arr;
```
- Traverse the array using only pointer arithmetic (no `arr[i]`)
- Explain to yourself: why does `p + 1` advance by 4 bytes, not 1?

**1.3 Double pointers**
```c
void update(int **ptr, int *new_val);
```
- Write a function that takes a `int **` and reassigns where it points
- Use case: this pattern appears in linked lists and dynamic buffer management

**1.4 Pointer to void**
- Write a function `void print_bytes(void *ptr, size_t len)` that prints the raw bytes of any variable
- Call it on an `int`, a `float`, and a `struct`
- This is how `memcpy` and similar functions work internally

---

## Task 2 — Memory Management (1 hr)

**Goal:** Understand stack vs heap and when each is appropriate. In embedded work, heap use is often avoided — you need to know why.

### Exercises

**2.1 Stack vs heap**
- Allocate an `int` on the stack; allocate one on the heap with `malloc`
- Print both addresses — observe the difference in address ranges
- Free the heap allocation; observe what happens if you forget (use Valgrind if available)

**2.2 Dynamic array**
- Write a function that allocates an array of `n` integers on the heap, fills it with values, and returns the pointer
- Caller is responsible for freeing — implement this correctly

**2.3 Why embedded avoids heap**

Write a short comment block in your code (3–5 sentences) explaining:
- What heap fragmentation is
- Why `malloc` is non-deterministic in timing
- What the embedded alternative is (static allocation, memory pools)

This is interview-relevant — you should be able to say this out loud.

---

## Task 3 — Structs (1 hr)

**Goal:** Use structs the way firmware code does — packed, typed, and with careful attention to memory layout.

### Exercises

**3.1 Basic struct**
```c
typedef struct {
    uint8_t  id;
    uint16_t value;
    uint8_t  status;
} SensorReading;
```
- Print `sizeof(SensorReading)` — is it what you expect? Why not?
- Look up struct padding and alignment rules

**3.2 Packed struct**
```c
typedef struct __attribute__((packed)) {
    uint8_t  id;
    uint16_t value;
    uint8_t  status;
} SensorReadingPacked;
```
- Print `sizeof(SensorReadingPacked)` — compare to above
- When would you use `packed`? (hint: network packets, hardware register maps)

**3.3 Nested structs and pointers to structs**
- Define a `Device` struct that contains a `SensorReading` and a `char name[16]`
- Write a function `void update_sensor(Device *dev, uint16_t new_val)` that modifies the reading through a pointer
- Pass by pointer, not by value — understand why this matters for large structs

---

## Task 4 — Function Pointers (1.5 hrs)

**Goal:** Function pointers are used everywhere in embedded firmware — ISR tables, state machines, driver interfaces, callbacks. Get fluent with the syntax.

### Exercises

**4.1 Basic function pointer**
```c
void say_hello(void) { printf("Hello\n"); }
void (*fp)(void) = say_hello;
fp();
```
- Write two functions with the same signature; store both in a function pointer; call each

**4.2 Function pointer as parameter (callback)**
```c
void process(int *arr, size_t len, void (*callback)(int));
```
- Implement `process` — it calls `callback` on each element
- Pass two different callbacks: one that prints, one that doubles the value in place

**4.3 Array of function pointers (dispatch table)**
```c
void (*handlers[4])(void) = { handler0, handler1, handler2, handler3 };
handlers[event_id]();
```
- Define 4 handler functions
- Call the correct one based on an index
- This is how interrupt vector tables and command parsers work

**4.4 typedef for function pointers**
```c
typedef void (*EventHandler)(uint8_t event_id, void *data);
```
- Rewrite your dispatch table using a typedef
- Makes the code much more readable — use this pattern going forward

---

## Task 5 — Linked List (1.5 hrs)

**Goal:** Implement a singly linked list from scratch. This is a classic interview exercise and also directly relevant to how embedded systems manage queues, buffers, and free lists.

### Implementation

```c
typedef struct Node {
    int data;
    struct Node *next;
} Node;
```

Implement the following functions:

| Function | Description |
|----------|-------------|
| `Node* node_create(int data)` | Allocate and return a new node |
| `void list_insert_head(Node **head, int data)` | Insert at front |
| `void list_insert_tail(Node **head, int data)` | Insert at back |
| `void list_delete(Node **head, int data)` | Delete first node with matching data |
| `void list_print(Node *head)` | Print all values |
| `void list_free(Node **head)` | Free all nodes; set head to NULL |

**Requirements:**
- Use `Node **head` (double pointer) for insert/delete — understand why
- Every `malloc` must have a corresponding `free`
- Run under Valgrind if available: `valgrind --leak-check=full ./your_program`

---

## Task 6 — State Machine with Function Pointers (1.5 hrs)

**Goal:** Implement a state machine using function pointers. State machines are one of the most common patterns in embedded firmware.

### Specification

Model a simple traffic light controller:

**States:** `STATE_RED`, `STATE_GREEN`, `STATE_YELLOW`  
**Events:** `EVENT_TIMER`, `EVENT_EMERGENCY`

**Transitions:**
```
RED    + TIMER      → GREEN
GREEN  + TIMER      → YELLOW
YELLOW + TIMER      → RED
ANY    + EMERGENCY  → RED
```

### Implementation requirements

```c
typedef enum { STATE_RED, STATE_GREEN, STATE_YELLOW, STATE_COUNT } State;
typedef enum { EVENT_TIMER, EVENT_EMERGENCY, EVENT_COUNT } Event;
typedef State (*StateHandler)(Event event);
```

- Each state is a function that takes an event and returns the next state
- Store handlers in a `StateHandler handlers[STATE_COUNT]` array
- `main` runs a loop: feed events, print state transitions
- No `switch` statement for dispatch — use the function pointer table

**Example output:**
```
[RED]    + TIMER     → GREEN
[GREEN]  + TIMER     → YELLOW
[YELLOW] + EMERGENCY → RED
```

---

## Wrap-up Checklist

Before committing:

- [ ] All 6 tasks compile with `-Wall -Wextra -Werror` and zero warnings
- [ ] No memory leaks (check with Valgrind or manually trace every malloc/free)
- [ ] Code is readable — variable names make sense, no magic numbers
- [ ] Add a `NOTES.md` or comment block: what was easy, what was rusty, what you looked up

---

## Commit

```bash
git add week0/session01/
git commit -m "week0/session01: pointers, memory, structs, function pointers, linked list, state machine"
git push
```

Commit message should be specific. If something broke and you fixed it, mention it:
```
week0/session01: fix double-free bug in list_delete; add Valgrind clean run
```

---

## What to Read Tonight (optional, 30 min)

- [C Pointer Basics — Beej's Guide](https://beej.us/guide/bgc/html/split/pointers.html)
- Review your state machine code — tomorrow you'll do bit manipulation and register emulation, which builds on the same struct + function pointer patterns
