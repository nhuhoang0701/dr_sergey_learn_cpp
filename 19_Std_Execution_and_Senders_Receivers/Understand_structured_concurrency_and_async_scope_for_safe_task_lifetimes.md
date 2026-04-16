# Understand structured concurrency and async_scope for safe task lifetimes

**Category:** std::execution & Senders/Receivers  
**Item:** #609  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

Structured concurrency ensures that child async operations cannot outlive their parent scope. `async_scope` enforces this by tracking spawned tasks and joining them on scope exit.

```cpp

Unstructured (dangerous):        Structured (safe):

std::thread t(work);             async_scope scope;
t.detach();  // fire-and-forget   scope.spawn(work);
// t might outlive main!          scope.join();  // blocks until work done
// accessing destroyed locals!    // all spawned tasks guaranteed complete

```

---

## Self-Assessment

### Q1: Spawn fire-and-forget tasks with `async_scope`

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <vector>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    exec::async_scope scope;
    std::vector<int> results(10, 0);

    // Spawn multiple fire-and-forget tasks:
    for (int i = 0; i < 10; ++i) {
        auto work = stdexec::schedule(sched)
            | stdexec::then([i, &results]() {
                results[i] = i * i;
                std::cout << "Task " << i << " on thread "
                          << std::this_thread::get_id() << '\n';
            });
        // spawn_future returns immediately:
        scope.spawn(std::move(work));
    }

    // CRITICAL: join blocks until ALL spawned tasks complete:
    stdexec::sync_wait(scope.on_empty());

    // Safe to access results — all tasks are done:
    for (int i = 0; i < 10; ++i)
        std::cout << "results[" << i << "] = " << results[i] << '\n';
}
// Output (order varies):
// Task 3 on thread 140001
// Task 0 on thread 140002
// ...
// results[0] = 0
// results[1] = 1
// results[9] = 81

```

### Q2: `async_scope` prevents dangling async operations

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <string>

void process_request(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    // Local data that must not be accessed after function returns:
    std::string local_data = "important data";

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&local_data]() {
            // Access local_data safely — scope guarantees
            // this completes before function returns:
            std::cout << "Processing: " << local_data << '\n';
        })
    );

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&local_data]() {
            std::cout << "Also using: " << local_data << '\n';
        })
    );

    // scope's destructor or on_empty() ensures ALL spawned tasks complete
    // BEFORE local_data goes out of scope:
    stdexec::sync_wait(scope.on_empty());

    // Now local_data can safely be destroyed.
    // Without async_scope, tasks might still reference local_data!
}

int main() {
    exec::static_thread_pool pool(4);
    process_request(pool);
    std::cout << "All done\n";
}

```

### Q3: `async_scope` vs `thread::detach` — why detach is unsafe

| Aspect | `std::thread::detach()` | `async_scope` |
| --- | --- | --- |
| Lifetime tracking | None | Tracks all spawned tasks |
| Join guarantee | No | Yes (`on_empty()`) |
| Access to locals | Dangling references! | Safe — scope joins first |
| Cancellation | Manual (error-prone) | Structured (stop tokens) |
| Exception safety | Exceptions in detached threads = `std::terminate` | Errors propagated via error channel |
| Resource cleanup | Hope for the best | Deterministic |

```cpp

// DANGEROUS: thread::detach()
void bad_example() {
    std::string data = "hello";

    std::thread t([&data]() {
        std::this_thread::sleep_for(std::chrono::seconds(1));
        std::cout << data << '\n';  // data is DESTROYED!
    });
    t.detach();  // fire and forget

}  // data destroyed here, t still running -> UNDEFINED BEHAVIOR

// SAFE: async_scope
void good_example(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;
    std::string data = "hello";

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&data]() {
            std::cout << data << '\n';  // SAFE: scope joins first
        })
    );

    stdexec::sync_wait(scope.on_empty());
    // data destroyed AFTER task completes
}

```

```cpp

Lifetime comparison:

detach():                      async_scope:
┌─ function ─────────────┐     ┌─ function ──────────────────┐
│ create data             │     │ create data                    │
│ thread.detach()         │     │ scope.spawn(work)              │
│ return (data destroyed) │     │ scope.on_empty() // wait       │
└────────────────────────┘     │ return (data destroyed safely) │
  thread still running! UB!     └───────────────────────────────┘

```

---

## Notes

- `async_scope::spawn()` is the only way to do fire-and-forget in structured concurrency.
- `on_empty()` returns a sender that completes when all spawned work is done.
- `request_stop()` on the scope cancels all spawned tasks cooperatively.
- async_scope is proposed in P3149 for C++26 standardization.
- Never use `thread::detach()` in production — prefer `jthread` or `async_scope`.
