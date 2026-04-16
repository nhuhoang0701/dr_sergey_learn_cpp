# Use async_scope for structured concurrency and lifetime management

**Category:** std::execution & Senders/Receivers  
**Item:** #530  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p3149r5.html>  

---

## Topic Overview

`async_scope` is the P2300 mechanism for fire-and-forget tasks with guaranteed lifetime safety. It tracks all spawned work and ensures completion before destruction.

| Operation | Purpose |
| --- | --- |
| `scope.spawn(sender)` | Launch fire-and-forget task |
| `scope.spawn_future(sender)` | Launch task, get a sender for its result |
| `scope.on_empty()` | Returns sender that completes when all tasks finish |
| `scope.request_stop()` | Cancel all spawned tasks |

---

## Self-Assessment

### Q1: Spawn fire-and-forget work and join before scope exits

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <mutex>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    std::mutex mtx;
    int total = 0;

    // Spawn fire-and-forget tasks:
    for (int i = 0; i < 20; ++i) {
        scope.spawn(
            stdexec::schedule(sched)
            | stdexec::then([i, &total, &mtx]() {
                std::lock_guard lk(mtx);
                total += i;
            })
        );
    }

    // Join: wait for ALL spawned tasks to complete:
    stdexec::sync_wait(scope.on_empty());

    // Safe to read total — all tasks done:
    std::cout << "Total: " << total << '\n';  // 190 (0+1+...+19)
}

```

### Q2: Why unstructured spawn (detach) causes bugs

```cpp

#include <thread>
#include <iostream>
#include <vector>

// Bug 1: Lifetime — dangling reference
void lifetime_bug() {
    std::vector<int> data = {1, 2, 3};
    std::thread t([&data]() {
        // data might be destroyed!
        for (int x : data) std::cout << x;  // UB!
    });
    t.detach();
}  // data destroyed, thread still running

// Bug 2: Cancellation — no way to stop detached thread
void cancellation_bug() {
    std::thread t([]() {
        while (true) {
            // Infinite loop — no way to cancel!
            heavy_work();
        }
    });
    t.detach();
    // Can't cancel t. It runs until process exits.
}

// Bug 3: Exception safety — unhandled exception in detached thread
void exception_bug() {
    std::thread t([]() {
        throw std::runtime_error("oops");
        // -> std::terminate! No way to catch.
    });
    t.detach();
}

// async_scope solves ALL three:
// 1. Lifetime: on_empty() guarantees all tasks complete before locals die
// 2. Cancellation: request_stop() cancels all spawned tasks via stop_token
// 3. Exceptions: errors propagate via error channel, not terminate

```

### Q3: `async_scope` propagates cancellation on destruction

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    std::atomic<int> completed{0};

    {
        exec::async_scope scope;

        // Spawn long-running tasks:
        for (int i = 0; i < 10; ++i) {
            scope.spawn(
                stdexec::schedule(sched)
                | stdexec::then([&completed, i]() {
                    // Simulate work that checks for cancellation:
                    // In real code, check get_stop_token().stop_requested()
                    completed.fetch_add(1);
                })
            );
        }

        // Request cancellation of all spawned work:
        scope.request_stop();

        // on_empty() waits for tasks that already started to finish,
        // but cancelled tasks complete with set_stopped():
        stdexec::sync_wait(scope.on_empty());

        // Some tasks may have completed before cancellation:
        std::cout << "Completed: " << completed.load() << '\n';
    }
    // scope destroyed safely — all work is done or cancelled
}

// Cancellation flow:
// 1. scope.request_stop() sets the stop source
// 2. Each spawned sender gets a stop_token from the scope
// 3. stop_requested() returns true
// 4. Well-behaved senders call set_stopped(receiver)
// 5. scope tracks all completions (value, error, or stopped)
// 6. on_empty() completes when count reaches zero

```

---

## Notes

- `async_scope` is proposed in P3149 (counting_scope) for C++26.
- `spawn()` is fire-and-forget; `spawn_future()` returns a sender for the result.
- Always call `on_empty()` or let the scope destructor handle cleanup.
- `request_stop()` is cooperative — senders must check stop_token to actually cancel.
- Think of `async_scope` as a `WaitGroup` (Go) or `TaskGroup` (Swift) for C++.
