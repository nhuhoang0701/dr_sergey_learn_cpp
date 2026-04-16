# Understand structured concurrency with async_scope

**Category:** std::execution & Senders/Receivers  
**Item:** #709  
**Standard:** C++11  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

This topic focuses on practical patterns with `async_scope`: spawning child senders, preventing dangling work, and the comparison with detached threads. `async_scope` enforces that parent outlives children.

```cpp

async_scope lifecycle:

  scope created
       │
  spawn(sender1)  spawn(sender2)  spawn(sender3)
       │               │               │
       └───────────────┼───────────────┘
                       │
               on_empty() → waits until ALL done
                       │
               scope destroyed safely

```

---

## Self-Assessment

### Q1: Spawn child senders and wait before scope destruction

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    std::atomic<int> counter{0};

    // Spawn 5 child senders:
    for (int i = 0; i < 5; ++i) {
        scope.spawn(
            stdexec::schedule(sched)
            | stdexec::then([i, &counter]() {
                // Simulate varying work duration:
                std::this_thread::sleep_for(
                    std::chrono::milliseconds(10 * (5 - i)));
                counter.fetch_add(1, std::memory_order_relaxed);
                std::cout << "Child " << i << " done\n";
            })
        );
    }

    std::cout << "All children spawned, waiting...\n";

    // Wait for ALL children to complete:
    stdexec::sync_wait(scope.on_empty());

    // All children guaranteed complete:
    std::cout << "Counter: " << counter.load() << '\n';  // always 5
}
// Output:
// All children spawned, waiting...
// Child 4 done
// Child 3 done
// Child 2 done
// Child 1 done
// Child 0 done
// Counter: 5

```

### Q2: Scope prevents dangling: work completes before scope exits

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/async_scope.hpp>
#include <iostream>
#include <memory>

struct Resource {
    std::string name;
    ~Resource() { std::cout << "~Resource(" << name << ")\n"; }
};

void process(exec::static_thread_pool& pool) {
    auto sched = pool.get_scheduler();
    exec::async_scope scope;

    // Resource with a limited lifetime:
    auto resource = std::make_unique<Resource>(Resource{"database"});

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&resource]() {
            // Safe: scope guarantees this completes
            // before resource is destroyed
            std::cout << "Using: " << resource->name << '\n';
        })
    );

    scope.spawn(
        stdexec::schedule(sched)
        | stdexec::then([&resource]() {
            std::cout << "Also using: " << resource->name << '\n';
        })
    );

    // on_empty() ensures all spawned work is complete:
    stdexec::sync_wait(scope.on_empty());

    // NOW resource can safely be destroyed (unique_ptr dtor runs)
}
// Output:
// Using: database
// Also using: database
// ~Resource(database)   <-- destroyed AFTER all tasks complete

int main() {
    exec::static_thread_pool pool(4);
    process(pool);
    std::cout << "Done\n";
}

```

### Q3: `async_scope` vs detached `std::thread`

```cpp

#include <iostream>
#include <thread>
#include <chrono>
#include <vector>

// ===== DANGEROUS: detached threads =====
void bad_pattern() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    std::thread t([&data]() {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        // DANGER: data might be destroyed already!
        for (int x : data) std::cout << x << ' ';
    });
    t.detach();
    // data destroyed here -> thread has dangling reference
}

// ===== SAFE: async_scope =====
// void good_pattern(exec::static_thread_pool& pool) {
//     auto sched = pool.get_scheduler();
//     exec::async_scope scope;
//     std::vector<int> data = {1, 2, 3, 4, 5};
//
//     scope.spawn(stdexec::schedule(sched)
//         | stdexec::then([&data]() {
//             for (int x : data) std::cout << x << ' ';
//         }));
//
//     stdexec::sync_wait(scope.on_empty());
//     // data destroyed here -> AFTER task completes, SAFE
// }

```

| Problem | `thread::detach()` | `async_scope` |
| --- | --- | --- |
| Dangling references | Yes! No lifetime tracking | No — scope joins first |
| Resource leaks | Thread runs after cleanup | Tasks complete before cleanup |
| Unhandled exceptions | `std::terminate` | Error channel propagation |
| Cancellation | Ad-hoc, manual | Built-in stop_token |
| Debugging | Hard (no ownership) | Clear parent-child relationship |

```cpp

Rule of thumb:
  ✘ Never use thread::detach()
  ✘ Never use raw new + callback (same problem)
  ✔ Use jthread (C++20) for simple cases
  ✔ Use async_scope for fire-and-forget in sender pipelines
  ✔ Use when_all for fan-out with results

```

---

## Notes

- `async_scope::spawn()` takes ownership of the sender and manages the operation state.
- `on_empty()` is idempotent — can be called multiple times safely.
- `request_stop()` cancels all spawned-but-not-yet-completed tasks.
- In stdexec, `exec::async_scope` is the implementation; C++26 aims for `std::execution::async_scope`.
- Structured concurrency is a paradigm shift: ALL async work has clear ownership.
