# Use stdexec::split to multicast a sender to multiple receivers

**Category:** std::execution & Senders/Receivers  
**Item:** #710  
**Standard:** C++26  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

By default, senders are single-shot: they can only be connected to one receiver. `split()` makes a sender multi-shot by memoizing the result and sharing it with multiple downstream consumers.

```cpp

Without split:              With split:

sender ─→ one consumer      sender ── split() ─┬─→ consumer A
                                              ├─→ consumer B
                                              └─→ consumer C
                             executes ONCE, result shared

```

---

## Self-Assessment

### Q1: Share an expensive result with two pipelines

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int heavy_computation() {
    std::cout << "[expensive] Computing on thread "
              << std::this_thread::get_id() << '\n';
    return 42;
}

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Expensive computation that we want to run ONCE:
    auto expensive = stdexec::schedule(sched)
        | stdexec::then([]() { return heavy_computation(); })
        | stdexec::split();  // make it multi-shot

    // Consumer A: uses the result
    auto pipeline_a = expensive
        | stdexec::then([](int x) {
            std::cout << "Consumer A: " << x * 2 << '\n';  // 84
            return x * 2;
        });

    // Consumer B: uses the SAME result (no re-computation)
    auto pipeline_b = expensive
        | stdexec::then([](int x) {
            std::cout << "Consumer B: " << x + 10 << '\n';  // 52
            return x + 10;
        });

    // Run both consumers:
    auto combined = stdexec::when_all(
        std::move(pipeline_a),
        std::move(pipeline_b)
    );

    auto [a, b] = stdexec::sync_wait(std::move(combined)).value();
    std::cout << "A=" << a << ", B=" << b << '\n';
}
// Output:
// [expensive] Computing on thread 140001  (runs ONCE)
// Consumer A: 84
// Consumer B: 52
// A=84, B=52

```

### Q2: `split` memoizes — upstream runs only once

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> call_count{0};

int main() {
    auto sender = stdexec::just()
        | stdexec::then([]() {
            int n = ++call_count;  // count invocations
            std::cout << "Executed! Count: " << n << '\n';
            return 42;
        })
        | stdexec::split();  // memoize

    // Connect to 3 consumers:
    auto a = sender | stdexec::then([](int x) { return x + 1; });
    auto b = sender | stdexec::then([](int x) { return x + 2; });
    auto c = sender | stdexec::then([](int x) { return x + 3; });

    auto [ra, rb, rc] = stdexec::sync_wait(
        stdexec::when_all(
            std::move(a), std::move(b), std::move(c)
        )
    ).value();

    std::cout << "Results: " << ra << ", " << rb << ", " << rc << '\n';
    std::cout << "Call count: " << call_count.load() << '\n';
}
// Output:
// Executed! Count: 1           <-- only ONCE
// Results: 43, 44, 45
// Call count: 1

```

### Q3: Reference counting in `split`

`split()` creates a shared state with reference counting:

```cpp

Internal mechanics:

  split() creates:
  ┌─────────────────────────────────┐
  │ Shared State (heap-allocated)    │
  │  - upstream_op_state             │
  │  - memoized_value (or error)     │
  │  - ref_count: atomic<int>        │
  │  - completion_status              │
  │  - list of waiting receivers      │
  └────────────────┬────────────────┘
                 │
  ┌─────────────┼─────────────┐
  ▼              ▼              ▼
  copy 1         copy 2         copy 3
  (ref_count=3)  (ref_count=3)  (ref_count=3)
  │              │              │
  connect(A)     connect(B)     connect(C)
  │              │              │
  start(A)  ── triggers upstream start (first one)
  start(B)  ── waits for memoized result
  start(C)  ── waits for memoized result

```

**Lifecycle:**

1. `split()` allocates shared state with `ref_count = 0`.
2. Each copy of the split sender increments `ref_count`.
3. First `start()` triggers the upstream sender.
4. When upstream completes, result is memoized.
5. All waiting receivers get the memoized result.
6. When last receiver completes and last copy is destroyed, shared state is freed.

```cpp

// split() is the ONLY sender adaptor that allocates heap memory.
// This is necessary because:
// 1. Multiple receivers need the same result
// 2. The result must outlive any single receiver
// 3. Thread-safe access requires shared ownership

// Without split, copying a sender and connecting it twice
// would execute the upstream work TWICE.

```

---

## Notes

- `split()` is the only standard sender adaptor that does heap allocation.
- The split sender is copyable (shared_ptr-like semantics).
- `ensure_started()` is similar but starts the upstream sender eagerly.
- Use `split()` when multiple consumers need the same expensive result.
- If you only need one consumer, avoid `split()` — unnecessary overhead.
