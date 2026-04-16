# Use std::execution::let_value, let_error, and let_stopped for dependent senders

**Category:** std::execution & Senders/Receivers  
**Item:** #524  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

The `let_*` family handles all three completion channels, each returning a sender for the next async step:

| Adaptor | Triggered by | Callback signature | Returns |
| --- | --- | --- | --- |
| `let_value(f)` | `set_value(vals...)` | `f(vals...)` | sender |
| `let_error(f)` | `set_error(err)` | `f(err)` | sender |
| `let_stopped(f)` | `set_stopped()` | `f()` | sender |

```cpp

Completion channels:

  sender ─┬─ set_value  ─→ let_value(f)  ─→ f(vals...) ─→ sender
         ├─ set_error  ─→ let_error(f)  ─→ f(err)     ─→ sender
         └─ set_stopped ─→ let_stopped(f) ─→ f()       ─→ sender

```

---

## Self-Assessment

### Q1: `let_value` — type-dependent chaining

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <string>
#include <variant>

// The power of let_value: the RETURNED SENDER TYPE
// can depend on the runtime value from the predecessor.

auto process_command(int cmd) {
    if (cmd == 1) {
        // Returns a sender<string>:
        return stdexec::just(std::string("processed"));
    } else {
        // Returns the SAME sender type:
        return stdexec::just(std::string("unknown command"));
    }
}

int main() {
    // let_value: next step DEPENDS on the result
    auto pipeline = stdexec::just(1)
        | stdexec::let_value([](int cmd) {
            std::cout << "Command: " << cmd << '\n';
            return process_command(cmd);  // returns a SENDER
        })
        | stdexec::then([](const std::string& result) {
            std::cout << "Result: " << result << '\n';
            return result;
        });

    auto [val] = stdexec::sync_wait(std::move(pipeline)).value();
    // Output:
    // Command: 1
    // Result: processed

    // Nested let_value for multi-step dependent chains:
    auto deep = stdexec::just(10)
        | stdexec::let_value([](int x) {
            return stdexec::just(x * 2)
                | stdexec::let_value([](int y) {
                    return stdexec::just(y + 5);  // nested!
                });
        });
    auto [r] = stdexec::sync_wait(std::move(deep)).value();
    std::cout << "Deep: " << r << '\n';  // 25
}

```

### Q2: `let_value` is monadic bind (flatMap)

```cpp

Monad parallel:

  Haskell:    m a >>= (\a -> m b)       // bind
  Rust:       future.and_then(|a| ...)
  JavaScript: promise.then(a => ...)
  P2300:      sender | let_value([](a) { return sender; })

Laws:

  1. Left identity:  just(x) | let_value(f)  ===  f(x)
  2. Right identity: s | let_value(just)      ===  s
  3. Associativity:  (s|let_value(f)) | let_value(g)

                     === s | let_value([](x){ return f(x)|let_value(g); })

```

```cpp

// Why "monadic"?
// then(f):       Sender<A> -> (A -> B) -> Sender<B>           // functor map
// let_value(f):  Sender<A> -> (A -> Sender<B>) -> Sender<B>   // monad bind

// The difference: let_value FLATTENS the nested sender.
// Without let_value, then would give Sender<Sender<B>> which is unusable.

#include <stdexec/execution.hpp>
#include <iostream>

auto double_async(int x) {
    return stdexec::just(x * 2);  // returns a sender
}

int main() {
    // then would fail here (can't unwrap a sender-of-sender):
    // auto bad = just(21) | then(double_async);  // type: sender<sender<int>>!

    // let_value flattens:
    auto good = stdexec::just(21)
        | stdexec::let_value([](int x) {
            return double_async(x);  // sender<int>, not sender<sender<int>>
        });
    auto [val] = stdexec::sync_wait(std::move(good)).value();
    std::cout << val << '\n';  // 42
}

```

### Q3: `let_error` for retry on failure

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <atomic>

std::atomic<int> attempt_count{0};

auto unreliable_operation() {
    return stdexec::just()
        | stdexec::then([]() -> int {
            int n = ++attempt_count;
            if (n < 3) {
                throw std::runtime_error("Attempt " + std::to_string(n) + " failed");
            }
            return 42;  // success on 3rd try
        });
}

// Build a retry combinator using let_error:
auto with_retry(auto sender, int max_retries) {
    auto result = std::move(sender);
    for (int i = 0; i < max_retries; ++i) {
        result = std::move(result)
            | stdexec::let_error([](std::exception_ptr ep) {
                try { std::rethrow_exception(ep); }
                catch (const std::exception& e) {
                    std::cout << "Retrying after: " << e.what() << '\n';
                }
                return unreliable_operation();  // retry!
            });
    }
    return result;
}

int main() {
    auto pipeline = with_retry(unreliable_operation(), 3);

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Success: " << result << '\n';
    std::cout << "Total attempts: " << attempt_count.load() << '\n';
}
// Output:
// Retrying after: Attempt 1 failed
// Retrying after: Attempt 2 failed
// Success: 42
// Total attempts: 3

```

**`let_stopped` for cancellation recovery:**

```cpp

auto pipeline = some_cancellable_sender()
    | stdexec::let_stopped([]() {
        std::cout << "Operation was cancelled, using fallback...\n";
        return stdexec::just(default_value);  // recover from cancel
    });

```

---

## Notes

- `let_value` keeps predecessor values alive for the returned sender's lifetime.
- `let_error` only fires on error; values pass through unchanged.
- `let_stopped` only fires on cancellation; values and errors pass through.
- All three return senders, enabling complex async decision trees.
- For simple transforms (non-sender return), prefer `then`/`upon_error`/`upon_stopped`.
