# Understand the sender/receiver execution model (P2300)

**Category:** std::execution & Senders/Receivers  
**Item:** #601  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

P2300 defines three core roles that separate concerns in async programming:

| Role | Responsibility | Analogy |
| --- | --- | --- |
| **Sender** | Describes work to be done | A recipe (not cooked yet) |
| **Receiver** | Handles the result/error/cancellation | A callback with 3 channels |
| **Operation State** | Running work binding sender+receiver | A cooking session |

```cpp

P2300 protocol:

  Sender + Receiver ────connect()────→ Operation State
                                          │
                                      start()
                                          │
                                 (work happens)
                                          │
                     ┌────────────┼────────────┐
                     ▼            ▼            ▼
             set_value(recv)  set_error(recv)  set_stopped(recv)

```

---

## Self-Assessment

### Q1: Three roles in P2300

**Sender** (describes work):

- A sender is a lazy description of async work. It holds the parameters and logic but does NOT execute.
- Examples: `just(42)`, `schedule(sched) | then(f)`, custom I/O senders.
- Composable via pipe operator: `just(42) | then(f) | then(g)`.

**Receiver** (handles completion):

- A receiver is the "continuation" that processes the result. It has three methods:
  - `set_value(args...)` — success
  - `set_error(err)` — failure
  - `set_stopped()` — cancellation
- Also carries an **environment** (stop_token, scheduler, allocator).

**Operation State** (running work):

- Created by `connect(sender, receiver)`.
- `start(op_state)` begins execution.
- Must remain alive until one completion signal fires.
- Non-movable after `start()`.

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

// Minimal demonstration of the three roles:
struct my_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int)
    >;
    int value;

    template <typename Receiver>  // <-- Receiver role
    struct op_state {             // <-- Operation State role
        using operation_state_concept = stdexec::operation_state_t;
        int value;
        Receiver recv;
        void start() noexcept {
            stdexec::set_value(std::move(recv), value);
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return op_state<Receiver>{value, std::move(recv)};
    }
};  // <-- Sender role

int main() {
    auto [result] = stdexec::sync_wait(my_sender{42}).value();
    std::cout << result << '\n';  // 42
}

```

### Q2: Separation of what/where/result

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // WHAT to do (sender — describes computation):
    auto computation = stdexec::just(21)
        | stdexec::then([](int x) { return x * 2; });

    // WHERE to run it (scheduler — execution context):
    auto on_pool = stdexec::starts_on(sched, std::move(computation));

    // WHAT to do with result (receiver — handled by sync_wait):
    auto [result] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "Result: " << result << '\n';  // 42

    // Key insight: the SAME computation can run on DIFFERENT schedulers
    // without changing the computation code!
}

```

```cpp

Separation of concerns:

  WHAT to do          WHERE to run       WHAT to do with result
  (sender)            (scheduler)        (receiver/consumer)
  ┌─────────────┐    ┌───────────┐    ┌──────────────┐
  │ just(21)      │    │ thread pool │    │ sync_wait()    │
  │ | then(f)     │    │ GPU         │    │ | then(print)  │
  │ | then(g)     │    │ io_context  │    │ | store(var)   │
  └─────────────┘    └───────────┘    └──────────────┘
  Independent!       Independent!       Independent!

```

### Q3: P2300 vs `std::async` vs raw threads

| Aspect | Raw threads | `std::async` | P2300 senders |
| --- | --- | --- | --- |
| Composition | Manual join/detach | Limited (no chaining) | Pipe `\|` operator |
| Error handling | `std::terminate` | Exception from `get()` | Error channel |
| Cancellation | Manual signals | None built-in | Stop tokens |
| Scheduler control | Thread per task | Unspecified | Explicit schedulers |
| Resource management | Manual | Future owns result | Operation state RAII |
| Structured lifetime | No (detach is UB-prone) | Blocking `get()` | `async_scope` |

```cpp

// Raw threads: manual, error-prone
std::thread t([] { return heavy_work(); });
t.join(); // must not forget!

// std::async: simple but limited
auto f = std::async(std::launch::async, heavy_work);
int result = f.get(); // blocks, no chaining

// P2300: composable, structured, schedulable
auto pipeline = stdexec::schedule(sched)
    | stdexec::then(heavy_work)
    | stdexec::then(process_result)
    | stdexec::upon_error(handle_error);
auto [result] = stdexec::sync_wait(std::move(pipeline)).value();

```

---

## Notes

- P2300 is the biggest addition to C++ async since coroutines in C++20.
- The reference implementation is `stdexec` from NVIDIA (open source, MIT license).
- Key design goals: no allocations, no type erasure, zero-overhead abstraction.
- `sync_wait` is the only blocking primitive — the rest is fully lazy.
- Think of senders as `std::ranges` for async: lazy, composable, and value-semantic.
