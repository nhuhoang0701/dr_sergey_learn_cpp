# Use std::execution::split to multicast a sender to multiple receivers

**Category:** std::execution & Senders/Receivers  
**Item:** #610  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`split()` converts a single-shot sender into a multi-shot sender by caching (memoizing) the result. The upstream computation runs exactly once, and any number of downstream consumers share the cached result.

```cpp

Without split:                With split:
  sender ─→ one consumer       sender | split() ─┬─→ consumer A
                                              ├─→ consumer B
                                              └─→ consumer C
                               (runs ONCE, result shared)

```

---

## Self-Assessment

### Q1: Share an expensive computation with two downstream consumers

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Expensive computation:
    auto expensive = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "[expensive] Running on thread "
                      << std::this_thread::get_id() << '\n';
            // Simulate heavy work:
            return 42;
        })
        | stdexec::split();  // now it's multi-shot

    // Consumer 1: doubles the result
    auto consumer_a = expensive
        | stdexec::then([](int x) {
            std::cout << "Consumer A: " << x * 2 << '\n';
            return x * 2;
        });

    // Consumer 2: adds to the result
    auto consumer_b = expensive
        | stdexec::then([](int x) {
            std::cout << "Consumer B: " << x + 100 << '\n';
            return x + 100;
        });

    // Run both concurrently:
    auto [a, b] = stdexec::sync_wait(
        stdexec::when_all(std::move(consumer_a), std::move(consumer_b))
    ).value();

    std::cout << "A=" << a << ", B=" << b << '\n';  // A=84, B=142
}
// Output:
// [expensive] Running on thread 14000050  (runs ONCE)
// Consumer A: 84
// Consumer B: 142
// A=84, B=142

```

### Q2: `split()` caches the result — computation runs only once

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> execution_count{0};

int main() {
    auto sender = stdexec::just()
        | stdexec::then([]() {
            ++execution_count;  // count invocations
            std::cout << "Executed (count: " << execution_count.load() << ")\n";
            return 42;
        })
        | stdexec::split();  // cache result

    // Three independent consumers:
    auto a = sender | stdexec::then([](int x) { return x + 1; });
    auto b = sender | stdexec::then([](int x) { return x + 2; });
    auto c = sender | stdexec::then([](int x) { return x + 3; });

    auto [ra, rb, rc] = stdexec::sync_wait(
        stdexec::when_all(std::move(a), std::move(b), std::move(c))
    ).value();

    std::cout << "Results: " << ra << ", " << rb << ", " << rc << '\n';
    std::cout << "Executions: " << execution_count.load() << '\n';
}
// Output:
// Executed (count: 1)    <-- only ONCE despite 3 consumers
// Results: 43, 44, 45
// Executions: 1

```

**How caching works internally:**

- First `start()` triggers the upstream sender
- Result is stored in shared heap-allocated state
- Subsequent consumers receive the cached result immediately
- Error and stopped channels are also cached

### Q3: Practical use case — compute hash once, use in cache lookup + logging

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <string>
#include <functional>

std::size_t compute_hash(const std::string& data) {
    return std::hash<std::string>{}(data);
}

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // Hash computation (expensive, run once):
    auto hash_sender = stdexec::starts_on(sched,
        stdexec::just(std::string("important_document_content"))
        | stdexec::then([](const std::string& data) {
            std::cout << "Computing hash...\n";
            return compute_hash(data);
        })
    ) | stdexec::split();  // share result

    // Consumer 1: Cache lookup
    auto cache_lookup = hash_sender
        | stdexec::then([](std::size_t hash) {
            std::cout << "Cache lookup for hash: " << hash << '\n';
            bool found = false;  // simulated
            return found;
        });

    // Consumer 2: Audit log
    auto audit_log = hash_sender
        | stdexec::then([](std::size_t hash) {
            std::cout << "Audit log: document hash = " << hash << '\n';
            return true;  // logged successfully
        });

    // Consumer 3: Integrity verification
    auto verify = hash_sender
        | stdexec::then([](std::size_t hash) {
            std::cout << "Integrity check with hash: " << hash << '\n';
            return hash != 0;  // valid
        });

    auto [cached, logged, valid] = stdexec::sync_wait(
        stdexec::when_all(
            std::move(cache_lookup),
            std::move(audit_log),
            std::move(verify)
        )
    ).value();

    std::cout << "Cached: " << cached
              << ", Logged: " << logged
              << ", Valid: " << valid << '\n';
}
// Output:
// Computing hash...              (runs ONCE)
// Cache lookup for hash: 12345
// Audit log: document hash = 12345
// Integrity check with hash: 12345
// Cached: 0, Logged: 1, Valid: 1

```

---

## Notes

- `split()` is the only standard sender adaptor that uses heap allocation.
- The split sender is **copyable** (shared_ptr-like semantics).
- `ensure_started()` is like `split()` but eagerly starts the upstream.
- Error results are also cached and delivered to all consumers.
- Avoid `split()` when only one consumer exists — unnecessary overhead.
