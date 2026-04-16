# Use stdexec::when_all to join multiple senders

**Category:** std::execution & Senders/Receivers  
**Item:** #703  
**Standard:** C++26  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`when_all` launches multiple senders concurrently and joins their results. It enforces **structured concurrency**: the `when_all` sender does NOT complete until every child sender has finished (or been cancelled).

```cpp

Structured concurrency guarantee:

  when_all(A, B, C)
    │
    ├─ start A  ───────────────────────┐
    ├─ start B  ────────────┐          │
    └─ start C  ─────┐          │          │
                      │          │          │
                   C done     B done     A done
                      │          │          │
                      └──────────┼──────────┘
                                 │
                          when_all completes
                          (only after ALL done)

```

---

## Self-Assessment

### Q1: Join two independent async computations

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto task_a = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[A] Computing on " << std::this_thread::get_id() << '\n';
            return 100;
        }));

    auto task_b = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "[B] Computing on " << std::this_thread::get_id() << '\n';
            return std::string("hello");
        }));

    // when_all: both run concurrently, results collected:
    auto [num, str] = stdexec::sync_wait(
        stdexec::when_all(std::move(task_a), std::move(task_b))
    ).value();

    std::cout << "Num: " << num << ", Str: " << str << '\n';
}
// Output:
// [A] Computing on 14000001
// [B] Computing on 14000002
// Num: 100, Str: hello

```

### Q2: Error cancels remaining senders

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> completions{0};

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto good_task = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            ++completions;
            return 42;
        }));

    auto bad_task = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() -> int {
            throw std::runtime_error("bad task!");
        }));

    auto good_task2 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            ++completions;
            return 99;
        }));

    try {
        stdexec::sync_wait(
            stdexec::when_all(
                std::move(good_task),
                std::move(bad_task),
                std::move(good_task2)
            )
        );
    } catch (const std::runtime_error& e) {
        std::cout << "Error: " << e.what() << '\n';
    }

    // Some good tasks may have completed before cancellation was received.
    // But when_all waits for ALL to finish before propagating the error.
    std::cout << "Completions: " << completions.load() << '\n';
}
// Output:
// Error: bad task!
// Completions: may be 0, 1, or 2 (depends on timing)

```

### Q3: Structured concurrency — `when_all` never leaks tasks

**Structured concurrency** means:

1. Child lifetimes are bounded by the parent
2. `when_all` does NOT complete until ALL children finish
3. Even on error/cancellation, `when_all` waits for cleanup

```cpp

// SAFE: when_all guarantees all tasks complete before returning
auto pipeline = stdexec::when_all(
    stdexec::starts_on(sched, task_a),
    stdexec::starts_on(sched, task_b),
    stdexec::starts_on(sched, task_c)
);
auto result = stdexec::sync_wait(std::move(pipeline));
// Here, ALL tasks are done. No dangling threads.

// UNSAFE (old style): fire-and-forget with std::async
auto f1 = std::async(task_a);  // might outlive scope!
auto f2 = std::async(task_b);  // might outlive scope!
// If exception thrown before get(), futures destructor blocks.
// No structured lifetime guarantee.

```

```cpp

Structured vs Unstructured:

Structured (when_all):           Unstructured (async):
┌────────────────────┐     ┌────────────────────┐
│ scope {             │     │ scope {             │
│   when_all(A, B, C) │     │   async(A)          │
│   // ALL finish     │     │   async(B) // may    │
│ }                   │     │ } // leak if error   │
│ // safe: all done   │     │ // B might still run!│
└────────────────────┘     └────────────────────┘

```

---

## Notes

- `when_all` is the structured concurrency primitive of P2300.
- Results are in **declaration order**, not completion order.
- Use `when_all` + `let_value` for dependent parallel + sequential patterns.
- `when_all_with_variant` handles senders with different signatures.
- `when_all` with zero senders completes immediately with `set_value()`.
