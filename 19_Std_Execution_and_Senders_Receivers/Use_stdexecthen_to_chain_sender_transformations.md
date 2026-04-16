# Use stdexec::then to chain sender transformations

**Category:** std::execution & Senders/Receivers  
**Item:** #702  
**Standard:** C++20  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`then` is the most fundamental sender adaptor. It transforms the value produced by a sender through a callable, returning a new sender. No work executes until the pipeline is started.

| Property | Detail |
| --- | --- |
| Signature | `then(callable)` |
| Input | Value from predecessor sender |
| Output | New sender carrying `callable(values...)` |
| Laziness | No work until `connect` + `start` |
| Exception | If `callable` throws, error propagates via `set_error` |

```cpp

just(42) | then(double_it) | then(add_one)

Pipeline (lazy):   42  ───→  84  ───→  85
                        then       then
No work until sync_wait/start triggers it.

```

---

## Self-Assessment

### Q1: Build a simple `then` pipeline

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // Build a pipeline: just(42) | then(x*2) | then(x+1)
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Step 1: " << x << " * 2 = " << x * 2 << '\n';
            return x * 2;
        })
        | stdexec::then([](int x) {
            std::cout << "Step 2: " << x << " + 1 = " << x + 1 << '\n';
            return x + 1;
        });

    // Nothing has executed yet! The pipeline is lazy.
    std::cout << "Pipeline built (no work done yet)\n";

    // sync_wait drives the pipeline:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 85

    // Multi-value example:
    auto multi = stdexec::just(10, 20)
        | stdexec::then([](int a, int b) {
            return a + b;  // then receives ALL values from predecessor
        });
    auto [sum] = stdexec::sync_wait(std::move(multi)).value();
    std::cout << "Sum: " << sum << '\n';  // 30

    // Void return:
    auto logging = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Logging: " << x << '\n';  // returns void
        });
    stdexec::sync_wait(std::move(logging));  // completes with no value
}
// Output:
// Pipeline built (no work done yet)
// Step 1: 42 * 2 = 84
// Step 2: 84 + 1 = 85
// Result: 85
// Sum: 30
// Logging: 42

```

### Q2: `then` returns a new sender — no work until started

```cpp

Laziness model:

  auto s1 = just(42);                    // creates sender (no work)
  auto s2 = s1 | then([](int x){...});   // creates sender (no work)
  auto s3 = s2 | then([](int x){...});   // creates sender (no work)
  //
  // At this point, ZERO functions have been called.
  // s3 is just a description of work to do.
  //
  auto op = connect(s3, my_receiver);    // compile-time wiring
  start(op);                             // NOW work begins!
  //
  // sync_wait does connect + start internally.

```

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    int call_count = 0;

    auto pipeline = stdexec::just(42)
        | stdexec::then([&](int x) {
            ++call_count;
            return x * 2;
        });

    std::cout << "Calls after building: " << call_count << '\n';  // 0

    auto [r1] = stdexec::sync_wait(pipeline).value();  // pipeline is copyable? No!
    std::cout << "Calls after first run: " << call_count << '\n';  // 1
    std::cout << "Result: " << r1 << '\n';  // 84

    // Rebuild to run again (senders are move-only):
    auto pipeline2 = stdexec::just(42)
        | stdexec::then([&](int x) {
            ++call_count;
            return x * 2;
        });
    auto [r2] = stdexec::sync_wait(std::move(pipeline2)).value();
    std::cout << "Calls after second run: " << call_count << '\n';  // 2
}

```

### Q3: `then` propagates exceptions via the error channel

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <stdexcept>

int main() {
    // If the callable throws, the exception goes to set_error:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) -> int {
            if (x > 0) throw std::runtime_error("boom!");
            return x;
        })
        | stdexec::then([](int x) {
            // This is NEVER called if the previous then threw:
            std::cout << "This won't print\n";
            return x;
        });

    // sync_wait re-throws the exception:
    try {
        stdexec::sync_wait(std::move(pipeline));
    } catch (const std::runtime_error& e) {
        std::cout << "Caught: " << e.what() << '\n';  // Caught: boom!
    }

    // Use upon_error to handle errors in the pipeline:
    auto with_recovery = stdexec::just(42)
        | stdexec::then([](int x) -> int {
            throw std::runtime_error("fail");
        })
        | stdexec::upon_error([](std::exception_ptr ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cout << "Recovered from: " << e.what() << '\n';
            }
            return -1;  // fallback value
        });

    auto [val] = stdexec::sync_wait(std::move(with_recovery)).value();
    std::cout << "Recovered value: " << val << '\n';  // -1
}
// Output:
// Caught: boom!
// Recovered from: fail
// Recovered value: -1

```

```cpp

Error propagation flow:

  just(42) ─→ then(f1) ──── then(f2) ──── sync_wait
                │                              │
               throws              set_error ─→ rethrow
              exception                         as exception
                │
                └─── skips f2 ────────────┘
                     (error channel)

```

---

## Notes

- `then` is like a functor map over the value channel.
- `then` only operates on the value channel. Errors skip `then` nodes.
- `upon_error` is the error-channel equivalent of `then`.
- `upon_stopped` is the stopped-channel equivalent of `then`.
- `then` returns a move-only sender — use `split()` to share.
- Prefer `then` for synchronous transforms; use `let_value` when you need to return another sender.
