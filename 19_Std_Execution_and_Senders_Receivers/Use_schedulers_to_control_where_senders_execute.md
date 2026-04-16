# Use schedulers to control where senders execute

**Category:** std::execution & Senders/Receivers  
**Item:** #605  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Schedulers decouple **what** to compute from **where** to compute it. You obtain a scheduler from an execution context and use it to direct senders onto that context.

| API | Purpose |
| --- | --- |
| `schedule(sched)` | Create a sender that completes on the scheduler |
| `starts_on(sched, sender)` | Ensure sender starts on given scheduler |
| `continues_on(sched)` | Transfer continuation to a different scheduler |

---

## Self-Assessment

### Q1: Use `continues_on` (transfer) to move execution onto a scheduler

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto pool_sched = pool.get_scheduler();

    // Start with a value on the current thread:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Before transfer, thread: "
                      << std::this_thread::get_id() << '\n';
            return x;
        })
        // Transfer execution to the thread pool:
        | stdexec::continues_on(pool_sched)
        | stdexec::then([](int x) {
            std::cout << "After transfer, thread: "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84
}
// Output:
// Before transfer, thread: 140000001 (main)
// After transfer, thread: 140000050 (pool)
// Result: 84

```

### Q2: `schedule()` creates a sender on a specific scheduler

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // schedule(sched) produces a sender with no value.
    // It just "enters" the scheduler's context:
    auto work = stdexec::schedule(sched)
        | stdexec::then([]() {
            // This runs on a pool thread:
            std::cout << "On pool: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            // Still on pool thread (unless transferred):
            return x * 2;
        });

    auto [val] = stdexec::sync_wait(std::move(work)).value();
    std::cout << "Result: " << val << '\n';  // 84

    // Multiple schedulers:
    exec::static_thread_pool io_pool(1);
    auto io_sched = io_pool.get_scheduler();

    // Start on compute pool, transfer to I/O pool:
    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() { return heavy_computation(); })
        | stdexec::continues_on(io_sched)
        | stdexec::then([](auto result) { write_to_disk(result); });
}

```

### Q3: Scheduler (where) vs Sender (what)

| Concept | Scheduler (Where) | Sender (What) |
| --- | --- | --- |
| Represents | Execution context (threads, GPU) | Work to be done |
| Owns resources | No (lightweight handle) | No (lazy description) |
| Examples | Thread pool scheduler, io_context sched | `just(42)`, `then(f)` |
| Copyable | Yes (cheap) | Movable (owns state) |
| Lifetime tied to | Execution context | Pipeline composition |

```cpp

Separation:
                    Sender (WHAT)
                    ┌─────────────────┐
                    │ just(42)        │
                    │ | then(double)  │
                    │ | then(print)   │
                    └────────┬────────┘
                             │
                    starts_on(sched, sender)
                             │
                    Scheduler (WHERE)
                    ┌────────┴────────┐
                    │ thread_pool     │
                    │ GPU context     │
                    │ io_context      │
                    └─────────────────┘

```

The same sender can run on any scheduler:

```cpp

auto work = stdexec::just(42) | stdexec::then([](int x) { return x * 2; });

// Run on thread pool:
auto [r1] = stdexec::sync_wait(stdexec::starts_on(pool_sched, work)).value();

// Run on GPU:
auto [r2] = stdexec::sync_wait(stdexec::starts_on(gpu_sched, work)).value();

// Run inline:
auto [r3] = stdexec::sync_wait(work).value();

```

---

## Notes

- `continues_on` was previously called `transfer` in P2300 drafts.
- `starts_on` is equivalent to `schedule(sched) | let_value([s]{ return s; })`.
- A scheduler is equality-comparable: `pool.get_scheduler() == pool.get_scheduler()` is true.
- You can create custom schedulers for specialized contexts (FPGA, RDMA, etc.).
- The separation of what/where is what makes P2300 senders truly portable.
