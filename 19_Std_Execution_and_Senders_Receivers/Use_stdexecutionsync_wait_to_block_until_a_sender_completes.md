# Use std::execution::sync_wait to block until a sender completes

**Category:** std::execution & Senders/Receivers  
**Item:** #606  
**Standard:** C++23  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`sync_wait` drives a sender pipeline from synchronous code. It blocks the current thread until the sender completes via any channel (value, error, or stopped).

| Channel | `sync_wait` behavior |
| --- | --- |
| `set_value(vals...)` | Returns `optional<tuple<vals...>>` |
| `set_error(exception_ptr)` | Rethrows the exception |
| `set_stopped()` | Returns `nullopt` |

---

## Self-Assessment

### Q1: Basic `sync_wait` usage — extract results

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // Build pipeline:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) { return x + 1; });

    // sync_wait blocks and returns the result:
    auto result = stdexec::sync_wait(std::move(pipeline));
    // type: std::optional<std::tuple<int>>

    // Extract with structured bindings:
    auto [value] = result.value();
    std::cout << "Result: " << value << '\n';  // 43

    // One-liner pattern:
    auto [v] = stdexec::sync_wait(
        stdexec::just(10, 20)
        | stdexec::then([](int a, int b) { return a + b; })
    ).value();
    std::cout << "Sum: " << v << '\n';  // 30

    // Void sender:
    auto r = stdexec::sync_wait(stdexec::just());
    if (r.has_value()) {
        std::cout << "Void sender completed\n";
    }
}

```

### Q2: `sync_wait` as the escape hatch from async to sync

```cpp

Async world (lazy, composable)     Sync world (blocking, sequential)
  ┌──────────────────────┐     ┌──────────────────────┐
  │ just(42)               │     │ int main() {         │
  │ | then(f)              │     │                      │
  │ | let_value(g)         │     │   auto [v] =         │
  │ | continues_on(sched)  │ ─── │     sync_wait(pipe)  │
  │ | then(h)              │     │       .value();      │
  └──────────────────────┘     │   use(v);            │
          lazy pipeline            │ }                    │
                                   └──────────────────────┘
                                        sync_wait bridges

```

**Rules for using `sync_wait`:**

```cpp

// GOOD: at program boundaries
int main() {
    auto [r] = stdexec::sync_wait(build_pipeline()).value();
    return r;
}

void test_case() {
    auto [r] = stdexec::sync_wait(build_pipeline()).value();
    ASSERT_EQ(r, 42);
}

// BAD: inside a sender callback (deadlock risk!)
auto bad = stdexec::just(42)
    | stdexec::then([](int x) {
        // NEVER do this:
        auto [r] = stdexec::sync_wait(other_pipeline()).value();  // DEADLOCK!
        return r;
    });
// Use let_value instead for nested async work.

```

### Q3: Error and cancellation handling

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>
#include <string>

int main() {
    // Case 1: Value -> optional(tuple(value))
    {
        auto result = stdexec::sync_wait(stdexec::just(42));
        if (result) {
            auto [v] = *result;
            std::cout << "Value: " << v << '\n';  // 42
        }
    }

    // Case 2: Error -> rethrows exception
    {
        auto err_sender = stdexec::just(42)
            | stdexec::then([](int) -> int {
                throw std::runtime_error("computation failed");
            });

        try {
            stdexec::sync_wait(std::move(err_sender));
        } catch (const std::runtime_error& e) {
            std::cout << "Error: " << e.what() << '\n';
        }
    }

    // Case 3: Stopped -> nullopt
    {
        auto stopped = stdexec::just_stopped();
        auto result = stdexec::sync_wait(std::move(stopped));
        if (!result) {
            std::cout << "Cancelled (nullopt)\n";
        }
    }

    // Practical pattern with error handling:
    auto safe_pipeline = stdexec::just(42)
        | stdexec::then([](int x) -> int {
            if (x < 0) throw std::invalid_argument("negative");
            return x * 2;
        })
        | stdexec::upon_error([](std::exception_ptr) {
            return 0;  // default on error
        });

    auto [safe] = stdexec::sync_wait(std::move(safe_pipeline)).value();
    std::cout << "Safe: " << safe << '\n';  // 84
}
// Output:
// Value: 42
// Error: computation failed
// Cancelled (nullopt)
// Safe: 84

```

---

## Notes

- `sync_wait` creates an internal `run_loop` to drive execution.
- Formally in the standard: `std::this_thread::sync_wait`.
- `sync_wait_with_variant` handles senders with multiple completion signatures.
- Never call from within a sender pipeline — use `let_value` for nested async.
- The return type is always `optional<tuple<Values...>>`.
