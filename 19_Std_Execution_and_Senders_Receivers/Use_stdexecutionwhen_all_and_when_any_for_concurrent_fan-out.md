# Use std::execution::when_all and when_any for concurrent fan-out

**Category:** std::execution & Senders/Receivers  
**Item:** #526  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

`when_all` and `when_any` express concurrent fan-out patterns:

| Adaptor | Completes when | Values | Error behavior |
| --- | --- | --- | --- |
| `when_all(s1, s2, s3)` | ALL complete | Tuple of all values | Cancels remaining on first error |
| `when_any` (proposed) | FIRST completes | First value | Cancels remaining |

```cpp

when_all:                    when_any:
  ┌─ s1 ─┐                    ┌─ s1 ─┐
  ├─ s2 ─┼─ wait ALL ─→    ├─ s2 ─┼─ first wins ─→
  └─ s3 ─┘                    └─ s3 ─┘  (cancel rest)

```

---

## Self-Assessment

### Q1: Launch three senders concurrently with `when_all`

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Three independent computations:
    auto s1 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "Task 1 on " << std::this_thread::get_id() << '\n';
            return 10;
        }));

    auto s2 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "Task 2 on " << std::this_thread::get_id() << '\n';
            return 20;
        }));

    auto s3 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            std::cout << "Task 3 on " << std::this_thread::get_id() << '\n';
            return 30;
        }));

    // when_all: all three run concurrently, collect all results:
    auto combined = stdexec::when_all(
        std::move(s1), std::move(s2), std::move(s3)
    );

    auto [a, b, c] = stdexec::sync_wait(std::move(combined)).value();
    std::cout << "Results: " << a << ", " << b << ", " << c << '\n';
    std::cout << "Sum: " << (a + b + c) << '\n';
}
// Output (order may vary):
// Task 1 on 14000001
// Task 3 on 14000003
// Task 2 on 14000002
// Results: 10, 20, 30
// Sum: 60

```

### Q2: `when_all` cancels remaining senders on first error

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <atomic>

std::atomic<bool> task3_started{false};

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto s1 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            return 10;  // succeeds
        }));

    auto s2 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() -> int {
            throw std::runtime_error("task 2 failed!");  // ERROR
        }));

    auto s3 = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            task3_started = true;
            return 30;  // may be cancelled before running
        }));

    auto combined = stdexec::when_all(
        std::move(s1), std::move(s2), std::move(s3)
    );

    try {
        stdexec::sync_wait(std::move(combined));
    } catch (const std::runtime_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }

    // when_all requests cancellation of remaining senders
    // when one completes with an error.
    // Task 3 may or may not have started depending on timing.
}
// Output:
// Caught: task 2 failed!

```

**Error propagation rules:**

- If ANY sender completes with an error, `when_all` cancels the rest
- The first error is propagated to the downstream receiver
- If ANY sender completes with `set_stopped`, `when_all` cancels the rest and propagates stopped
- Only if ALL complete with values does `when_all` complete with values

### Q3: `when_any` — first successful completion

```cpp

// when_any is not yet in P2300 R7, but the pattern can be built:
// The idea: first sender to complete wins, others are cancelled.

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>

// Simulated when_any using split + custom logic:
// In practice, use a library implementation or P2300 extensions.

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Race pattern: simulate with when_all + stopped_as_optional
    // When a real when_any is available, it would look like:
    //
    // auto fastest = stdexec::when_any(
    //     fetch_from_server_a(),  // might be slow
    //     fetch_from_server_b(),  // might be fast
    //     fetch_from_cache()      // probably fastest
    // );
    // // First to complete wins; others cancelled.

    // For now, demonstrate the concept with when_all:
    auto fast = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            return std::string("fast result");
        }));

    auto slow = stdexec::starts_on(sched,
        stdexec::just() | stdexec::then([]() {
            return std::string("slow result");
        }));

    // when_all collects both (when_any would pick first):
    auto [a, b] = stdexec::sync_wait(
        stdexec::when_all(std::move(fast), std::move(slow))
    ).value();

    std::cout << "Got: " << a << " and " << b << '\n';
}

```

**`when_any` cancellation semantics (when available):**

| Event | when_any behavior |
| --- | --- |
| First value | Complete with that value, cancel others |
| First error | Cancel others, propagate error |
| All stopped | Complete with stopped |
| Some value, some error | First responder wins |

```cpp

when_any race:
  ┌─ server A (200ms) ─┐
  ├─ server B (50ms)  ─┼─→ B wins! Cancel A and cache
  └─ cache (10ms)     ─┘     ▼
                          cache wins! Cancel A and B

```

---

## Notes

- `when_all` is the primary concurrent composition primitive in P2300.
- Results are returned in the **declaration order**, not completion order.
- `when_all` with zero senders completes immediately with `set_value()`.
- `when_all_with_variant` handles senders with different completion signatures.
- `when_any` is not in P2300 R7 but is a common extension in stdexec.
- Combine `when_all` with `split` for diamond-shaped dependency graphs.
