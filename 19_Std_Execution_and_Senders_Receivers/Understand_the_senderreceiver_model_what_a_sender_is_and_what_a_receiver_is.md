# Understand the sender/receiver model: what a sender is and what a receiver is

**Category:** std::execution & Senders/Receivers  
**Item:** #701  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

A **sender** is a lazy description of async work. A **receiver** is the consumer that handles the three possible outcomes.

```cpp

Sender (describes):                Receiver (consumes):
┌─────────────────────┐          ┌───────────────────────┐
│ What to compute       │          │ set_value(args...)     │
│ Completion signatures │─connect─→│ set_error(err)         │
│ No execution yet      │          │ set_stopped()          │
└─────────────────────┘          │ Environment (stop_tok) │
                                   └───────────────────────┘

```

---

## Self-Assessment

### Q1: Sender = description, Receiver = callback with 3 channels

**Sender:**

- A sender is a **value type** that describes async work.
- It does NOT start execution on construction.
- It advertises what it can produce via `completion_signatures`.
- It can be composed: `sender1 | then(f) | then(g)` builds a new sender.

**Receiver:**

- A receiver is the **callback** that processes the result.
- It has three methods (one must be called exactly once):
  - `set_value(recv, args...)` — success path
  - `set_error(recv, err)` — error path
  - `set_stopped(recv)` — cancellation path
- It also provides an **environment** via `get_env(recv)` containing:
  - Stop token for cancellation
  - Current scheduler
  - Allocator

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

// Sender: describes work
auto make_computation() {
    return stdexec::just(10, 20)          // sender: emit (10, 20)
        | stdexec::then([](int a, int b) { // sender: transform
            return a + b;
        });
    // Returns a sender. No work done yet!
}

int main() {
    auto sender = make_computation();
    // sender is just a description — zero side effects

    // sync_wait creates a receiver internally:
    //   set_value(recv, 30)  → stores 30
    //   set_error(recv, err) → stores exception
    //   set_stopped(recv)    → returns nullopt
    auto result = stdexec::sync_wait(std::move(sender));
    if (result) {
        auto [val] = *result;
        std::cout << val << '\n';  // 30
    }
}

```

### Q2: Senders are lazy — no work until connected+started

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int side_effect_counter = 0;

int main() {
    // Creating a sender does NOT execute anything:
    auto sender = stdexec::just(42)
        | stdexec::then([](int x) {
            ++side_effect_counter;  // NOT called yet!
            std::cout << "Executing!\n";
            return x * 2;
        });

    std::cout << "Sender created. Counter: "
              << side_effect_counter << '\n';  // 0

    // Can create multiple senders without executing:
    auto sender2 = stdexec::just(100)
        | stdexec::then([](int x) { return x - 1; });

    std::cout << "Two senders, still nothing executed.\n";

    // Execution happens only when connected to receiver and started:
    auto [r1] = stdexec::sync_wait(std::move(sender)).value();
    std::cout << "After sync_wait. Counter: "
              << side_effect_counter << '\n';  // 1
    std::cout << "Result: " << r1 << '\n';     // 84
}
// Output:
// Sender created. Counter: 0
// Two senders, still nothing executed.
// Executing!
// After sync_wait. Counter: 1
// Result: 84

```

**Key insight:** This laziness enables:

- The scheduler to decide WHERE work runs.
- The compiler to optimize the entire pipeline at once.
- Multiple pipelines to be composed before any execution.

### Q3: Sender/receiver vs future vs callbacks vs coroutines

| Model | Eager/Lazy | Composable | Cancellation | Error Handling | Allocation |
| --- | --- | --- | --- | --- | --- |
| **Callbacks** | Eager | No (callback hell) | Manual | Manual | None |
| **std::future** | Eager | No (.then() missing) | None | Exception | Heap |
| **Coroutines** | Lazy | Sequential (co_await) | Manual (stop_token) | co_await + try/catch | Coroutine frame |
| **Senders** | Lazy | Yes (pipe `\|`) | Built-in (stopped channel) | Typed error channel | Stack (op_state) |

```cpp

// Callback style:
async_read(fd, buffer, [](int bytes, error_code ec) {
    if (ec) { handle_error(ec); return; }
    async_write(out, buffer, [](int, error_code ec2) {
        if (ec2) { handle_error(ec2); return; }
        // callback hell...
    });
});

// Future style:
auto f = async_read(fd, buffer);
auto result = f.get();  // blocks, can't chain

// Coroutine style:
task<void> do_io() {
    auto bytes = co_await async_read(fd, buffer);
    co_await async_write(out, buffer);
}

// Sender style:
auto pipeline = async_read_sender(fd, buffer)
    | then([](auto bytes, auto buf) { return buf; })
    | let_value([&](auto buf) { return async_write_sender(out, buf); })
    | upon_error([](auto err) { log(err); });
sync_wait(std::move(pipeline));

```

---

## Notes

- Senders unify all async patterns: I/O, timers, thread pools, GPU dispatch.
- The three-channel model (value/error/stopped) is inspired by Reactive Extensions (Rx).
- Receivers are usually created by library code (sync_wait, then, when_all), not by users.
- `completion_signatures` enable zero-cost type checking at compile time.
- Sender/receiver is MORE powerful than coroutines for parallel fan-out and scheduler control.
