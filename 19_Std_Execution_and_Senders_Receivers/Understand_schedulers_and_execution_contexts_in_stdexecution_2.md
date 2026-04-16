# Understand schedulers and execution contexts in std::execution

**Category:** std::execution & Senders/Receivers  
**Item:** #705  
**Standard:** C++11  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

This topic explores schedulers as sender factories and practical patterns for moving work between execution contexts (thread pools, GPU, I/O loops).

A scheduler satisfies:

```cpp

concept scheduler = requires(Sched s) {
    { stdexec::schedule(s) } -> sender;  // produces a sender
};
// schedule(s) returns a sender that completes (set_value()) on the context.
// No values are sent — it just "starts" you on the right context.

```

---

## Self-Assessment

### Q1: Scheduler as a factory for senders on a context

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    // The execution context owns threads:
    exec::static_thread_pool pool(4);

    // The scheduler is a FACTORY — it creates senders,
    // it does NOT own or manage threads:
    auto sched = pool.get_scheduler();

    // schedule(sched) creates a sender that:
    //  - Does no computation itself
    //  - Completes with set_value() on a pool thread
    //  - Is the "entry point" to run code on this context
    auto start_on_pool = stdexec::schedule(sched);

    // Chain computation after the schedule sender:
    auto work = std::move(start_on_pool)
        | stdexec::then([]() {
            std::cout << "1. On pool thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            std::cout << "2. Still on pool: "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    // sync_wait drives the pipeline from the calling thread:
    auto [result] = stdexec::sync_wait(std::move(work)).value();
    std::cout << "3. Back on main: "
              << std::this_thread::get_id()
              << ", result = " << result << '\n';
}
// Output:
// 1. On pool thread: 140700001
// 2. Still on pool: 140700001
// 3. Back on main: 140700000, result = 84

```

### Q2: `schedule(thread_pool_scheduler)` to move work onto a pool

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <vector>
#include <numeric>
#include <cmath>

double heavy_computation(int n) {
    double sum = 0;
    for (int i = 1; i <= n; ++i)
        sum += std::sqrt(static_cast<double>(i));
    return sum;
}

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Pattern: schedule(sched) | then(work)
    // Runs 'work' on a pool thread:
    auto task1 = stdexec::schedule(sched)
        | stdexec::then([]() {
            return heavy_computation(1'000'000);
        });

    auto task2 = stdexec::schedule(sched)
        | stdexec::then([]() {
            return heavy_computation(2'000'000);
        });

    // Run both tasks concurrently on the pool:
    auto parallel = stdexec::when_all(
        std::move(task1),
        std::move(task2)
    ) | stdexec::then([](double a, double b) {
        std::cout << "Results: " << a << ", " << b << '\n';
        return a + b;
    });

    auto [total] = stdexec::sync_wait(std::move(parallel)).value();
    std::cout << "Total: " << total << '\n';
}

```

### Q3: `continues_on` (formerly `transfer`) for mid-pipeline context switch

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool compute(4);
    exec::static_thread_pool io(1);
    auto compute_sched = compute.get_scheduler();
    auto io_sched = io.get_scheduler();

    auto pipeline =
        // Start on compute pool:
        stdexec::schedule(compute_sched)
        | stdexec::then([]() -> int {
            std::cout << "[compute] thread "
                      << std::this_thread::get_id() << '\n';
            return 42;  // CPU-heavy result
        })
        // Transfer to I/O pool for writing:
        | stdexec::continues_on(io_sched)
        | stdexec::then([](int result) -> int {
            std::cout << "[io] thread "
                      << std::this_thread::get_id() << '\n';
            // write result to file/network ...
            return result;
        })
        // Transfer back to compute pool:
        | stdexec::continues_on(compute_sched)
        | stdexec::then([](int result) {
            std::cout << "[compute again] thread "
                      << std::this_thread::get_id() << '\n';
            return result * 2;
        });

    auto [val] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << val << '\n';  // 84
}

```

```cpp

Context transitions:

schedule(compute)  then(work)  continues_on(io)  then(save)  continues_on(compute)  then(more)
[compute pool]     [compute]   ──transfer──>      [io pool]   ──transfer──>          [compute]

```

---

## Notes

- `continues_on(sched)` replaces the older `transfer(sched)` name from earlier P2300 drafts.
- `starts_on(sched, sender)` sets the initial context for a sender; `continues_on` changes it mid-pipeline.
- Schedulers are lightweight value types — copying a scheduler does NOT spawn threads.
- `run_loop` is a single-threaded scheduler useful for testing (runs work inline).
- Multiple schedulers from the same pool compare equal (`==`).
