# Use std::future and std::promise for async result passing

**Category:** Concurrency & Parallelism  
**Item:** #91  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/future>  

---

## Topic Overview

`std::promise` and `std::future` form a **one-shot channel** for passing a single value (or exception) from a producer thread to a consumer thread.

### How They Fit Together

```cpp

Producer thread:                     Consumer thread:
──────────────────                   ──────────────────
std::promise<int> prom;              std::future<int> fut = prom.get_future();
// ... compute ...                   // ... do other work ...
prom.set_value(42);                  int result = fut.get(); // blocks until value is set
//    └── writes into shared state ──────────┘

```

### Three Ways to Create a Future

| Creator | Description |
| --- | --- |
| `std::async(func)` | Launches task, returns `future<ReturnType>` |
| `promise.get_future()` | Manual push channel for raw `std::thread` |
| `packaged_task.get_future()` | Wraps callable, captures return value |

---

## Self-Assessment

### Q1: Use std::async to launch a task and retrieve the result with future::get

**Answer:**

```cpp

#include <future>
#include <iostream>
#include <vector>
#include <numeric>
#include <cmath>

// Expensive computation
double compute_pi(int terms) {
    double sum = 0.0;
    for (int k = 0; k < terms; ++k) {
        sum += (k % 2 == 0 ? 1.0 : -1.0) / (2.0 * k + 1.0);
    }
    return 4.0 * sum;
}

int main() {
    // === Basic: single async task ===
    auto fut = std::async(std::launch::async, compute_pi, 10'000'000);
    // ^^^^^^ returns std::future<double>
    // Task runs concurrently in another thread

    // Do other work while computation runs...
    std::cout << "Computing...\n";

    double pi = fut.get(); // blocks until result is ready
    std::cout << "Pi ≈ " << pi << "\n";

    // === Multiple parallel tasks ===
    std::vector<std::future<double>> futures;
    int chunk = 2'500'000;
    for (int i = 0; i < 4; ++i) {
        futures.push_back(std::async(std::launch::async,
            [i, chunk] {
                double sum = 0.0;
                int start = i * chunk;
                for (int k = start; k < start + chunk; ++k)
                    sum += (k % 2 == 0 ? 1.0 : -1.0) / (2.0 * k + 1.0);
                return sum;
            }));
    }

    // Collect results
    double total = 0.0;
    for (auto& f : futures)
        total += f.get(); // blocks on each, but they run in parallel

    std::cout << "Parallel Pi ≈ " << 4.0 * total << "\n";

    // Output:
    // Computing...
    // Pi ≈ 3.14159
    // Parallel Pi ≈ 3.14159
}

```

**Key point:** `std::async(launch::async, func, args...)` spawns a new thread (or equivalent), runs `func(args...)`, and stores the result in the shared state. `fut.get()` blocks until the result is available and returns it. `get()` can only be called **once** — for multiple readers, use `std::shared_future`.

### Q2: Show how to propagate an exception from a thread to the caller via promise::set_exception

**Answer:**

```cpp

#include <future>
#include <thread>
#include <iostream>
#include <stdexcept>

// Worker that might fail
void worker(std::promise<int> prom, bool should_fail) {
    try {
        if (should_fail) {
            throw std::runtime_error("Database connection failed");
        }
        // Simulate work
        int result = 42;
        prom.set_value(result); // success path
    } catch (...) {
        // Capture the current exception and propagate it through the promise
        prom.set_exception(std::current_exception());
        //                  ^^^^^^^^^^^^^^^^^^^^^^^^
        // std::current_exception() returns an exception_ptr
        // wrapping whatever exception was thrown
    }
}

int main() {
    // === Success case ===
    {
        std::promise<int> prom;
        auto fut = prom.get_future();

        std::thread t(worker, std::move(prom), false);

        try {
            int result = fut.get(); // returns 42
            std::cout << "Success: " << result << "\n";
        } catch (const std::exception& e) {
            std::cout << "Error: " << e.what() << "\n";
        }
        t.join();
    }

    // === Failure case ===
    {
        std::promise<int> prom;
        auto fut = prom.get_future();

        std::thread t(worker, std::move(prom), true);

        try {
            int result = fut.get();
            // ^^^ This RETHROWS the exception set by set_exception
            std::cout << "Success: " << result << "\n"; // never reached
        } catch (const std::runtime_error& e) {
            std::cout << "Caught from thread: " << e.what() << "\n";
        }
        t.join();
    }

    // === What happens if promise is destroyed without setting? ===
    {
        std::future<int> fut;
        {
            std::promise<int> prom;
            fut = prom.get_future();
            // prom destroyed without set_value or set_exception!
        }
        try {
            fut.get();
        } catch (const std::future_error& e) {
            std::cout << "Broken promise: " << e.what() << "\n";
            // e.code() == std::future_errc::broken_promise
        }
    }

    // Output:
    // Success: 42
    // Caught from thread: Database connection failed
    // Broken promise: ...
}

```

**Explanation:** `set_exception(std::current_exception())` captures the active exception and stores it in the shared state. When the consumer calls `get()`, the exception is rethrown in the consumer's thread. If the promise is destroyed without setting a value or exception, `get()` throws `std::future_error` with code `broken_promise`.

### Q3: Explain the difference between std::launch::async and std::launch::deferred

**Answer:**

```cpp

std::launch::async vs std::launch::deferred
════════════════════════════════════════════

┌────────────────┬──────────────────────┬──────────────────────┐
│                │ launch::async        │ launch::deferred     │
├────────────────┼──────────────────────┼──────────────────────┤
│ Execution      │ New thread (or       │ On calling thread    │
│                │ thread pool)         │ when get()/wait()    │
│                │                      │ is called            │
│ When starts    │ Immediately          │ Lazily (on get())    │
│ Thread used    │ Yes (always)         │ No (same thread)     │
│ wait_for()     │ Returns ready/timeout│ Always returns       │
│                │                      │ deferred             │
│ Parallelism    │ Yes                  │ No!                  │
│ Side effects   │ May interleave       │ Deterministic (like  │
│                │ with caller          │ a function call)     │
└────────────────┴──────────────────────┴──────────────────────┘

DEFAULT (no policy specified):
  std::async(func)  ≡  std::async(launch::async | launch::deferred, func)
  The implementation CHOOSES — you don't know which!
  → ALWAYS specify the policy explicitly.

```

```cpp

#include <future>
#include <thread>
#include <iostream>
#include <chrono>

using namespace std::chrono_literals;

int main() {
    auto work = [] {
        std::cout << "  Running on thread "
                  << std::this_thread::get_id() << "\n";
        std::this_thread::sleep_for(100ms);
        return 42;
    };

    std::cout << "Main thread: " << std::this_thread::get_id() << "\n\n";

    // === launch::async — runs concurrently ===
    std::cout << "launch::async:\n";
    {
        auto fut = std::async(std::launch::async, work);
        std::cout << "  (task is already running...)\n";
        std::cout << "  Result: " << fut.get() << "\n";
    }
    // Output: Running on thread 12345 (different from main)

    // === launch::deferred — runs lazily ===
    std::cout << "\nlaunch::deferred:\n";
    {
        auto fut = std::async(std::launch::deferred, work);
        std::cout << "  (task has NOT started yet)\n";
        // wait_for always returns deferred for deferred futures:
        auto st = fut.wait_for(0ms);
        std::cout << "  wait_for: "
                  << (st == std::future_status::deferred ? "deferred" : "other") << "\n";
        std::cout << "  Calling get()...\n";
        std::cout << "  Result: " << fut.get() << "\n";
        // ^^^ NOW it runs, on main thread
    }
    // Output: Running on thread [same as main]

    // === GOTCHA: default policy may cause bugs ===
    // auto fut = std::async(func); // might be deferred!
    // if (fut.wait_for(1s) == std::future_status::timeout) {
    //     // This NEVER executes if the task was deferred
    //     // because wait_for returns deferred, not timeout
    // }

    // === GOTCHA: async future blocks on destruction ===
    {
        std::cout << "\nDestructor blocking:\n";
        auto start = std::chrono::steady_clock::now();
        {
            // Returned future is TEMPORARY — destroyed immediately
            std::async(std::launch::async, [] {
                std::this_thread::sleep_for(200ms);
            });
            // ^^^ BLOCKS HERE until the async task completes!
            // The temporary future's destructor joins the thread
        }
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start);
        std::cout << "  Took " << ms.count() << "ms (blocked by destructor)\n";
    }
}

```

**Critical gotcha:** If you discard the future returned by `std::async(launch::async, ...)`, the destructor **blocks** until the task completes. This effectively makes `std::async(launch::async, fire_and_forget)` synchronous!

---

## Notes

- **`shared_future<T>`:** Created via `fut.share()`. Multiple threads can call `get()` on the same shared state. Useful for broadcasting a result.
- **`promise` is move-only:** Must be `std::move`d into a thread. Each promise has exactly one associated future.
- **`set_value` / `set_exception` once:** Calling either a second time throws `std::future_error` with `promise_already_satisfied`.
- **`promise<void>`:** For signaling without data — `prom.set_value()` with no argument.
- **Thread safety:** `get()` is safe to call from one thread. `set_value()` is safe to call from one thread. But `get()` and `set_value()` on the same future/promise from different threads is the intended use.
- Compile with `-std=c++11 -O2 -pthread`.
