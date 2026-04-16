# Understand cancellation in P2300 via stop tokens and stop callbacks

**Category:** std::execution & Senders/Receivers  
**Item:** #611  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

P2300 provides structured cancellation through stop tokens. Every receiver carries a stop token in its environment; senders check it to cooperatively cancel.

```cpp

Cancellation flow:

  Parent scope          Sender/Operation         Receiver
  request_stop() ──→  stop_token.stop_requested()  │
                        │                            │
                   if true:                          │
                     clean up  ───────────────────→ set_stopped()
                   if false:                         │
                     continue work ───────────────→ set_value(result)

```

| Mechanism | Purpose |
| --- | --- |
| `stop_token` | Polling: check `stop_requested()` periodically |
| `stop_callback` | Reactive: register callback invoked when stop is requested |
| `set_stopped()` | Signal the receiver that operation was cancelled |
| `upon_stopped(f)` | Pipeline adaptor to handle cancellation |

---

## Self-Assessment

### Q1: Poll `get_stop_token()` inside a long-running sender

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <vector>
#include <numeric>

// A sender that does work in chunks and checks for cancellation:
struct chunked_compute_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(long long),
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    int iterations;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        int iterations;
        Receiver receiver;

        void start() noexcept {
            try {
                // Get the stop token from the receiver's environment:
                auto token = stdexec::get_stop_token(
                    stdexec::get_env(receiver));

                long long result = 0;
                for (int i = 0; i < iterations; ++i) {
                    // Poll for cancellation periodically:
                    if (i % 1000 == 0 && token.stop_requested()) {
                        std::cout << "Cancelled at iteration " << i << '\n';
                        stdexec::set_stopped(std::move(receiver));
                        return;
                    }
                    result += static_cast<long long>(i) * i;  // heavy work
                }

                stdexec::set_value(std::move(receiver), result);
            } catch (...) {
                stdexec::set_error(std::move(receiver),
                                   std::current_exception());
            }
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{iterations, std::move(recv)};
    }
};

int main() {
    // Without cancellation — runs to completion:
    auto pipeline = chunked_compute_sender{1000000}
        | stdexec::then([](long long r) {
            std::cout << "Result: " << r << '\n';
            return r;
        })
        | stdexec::upon_stopped([]() -> long long {
            std::cout << "Was cancelled, returning 0\n";
            return 0;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << result << '\n';
}

```

### Q2: Register a `stop_callback` for async I/O cancellation

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <stop_token>

// A sender wrapping an async I/O operation that supports cancellation:
struct async_io_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::size_t),
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    int fd;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        int fd;
        Receiver receiver;

        using stop_token_t = stdexec::stop_token_of_t<
            stdexec::env_of_t<Receiver>>;

        // stop_callback registered when start() is called:
        std::optional<std::stop_callback<std::function<void()>>> cb;

        void start() noexcept {
            auto token = stdexec::get_stop_token(
                stdexec::get_env(receiver));

            // Already cancelled?
            if (token.stop_requested()) {
                stdexec::set_stopped(std::move(receiver));
                return;
            }

            // Register callback: when stop is requested, cancel the I/O
            cb.emplace(token, [this]() {
                // This fires asynchronously when stop is requested:
                cancel_io(fd);  // platform-specific: cancel pending I/O
                std::cout << "I/O cancelled via stop_callback\n";
            });

            // Submit async I/O (simplified):
            submit_async_read(fd, [this](int result, bool cancelled) {
                cb.reset();  // unregister callback
                if (cancelled) {
                    stdexec::set_stopped(std::move(receiver));
                } else if (result < 0) {
                    stdexec::set_error(std::move(receiver),
                        std::make_exception_ptr(
                            std::runtime_error("I/O error")));
                } else {
                    stdexec::set_value(std::move(receiver),
                                      static_cast<std::size_t>(result));
                }
            });
        }

        // Platform stubs:
        static void cancel_io(int) { /* cancel pending I/O */ }
        static void submit_async_read(int, auto&&) { /* submit I/O */ }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{fd, std::move(recv)};
    }
};

// Key pattern:
// 1. Extract stop_token from receiver's environment
// 2. Register stop_callback that cancels platform-specific operation
// 3. When callback fires, clean up and call set_stopped()
// 4. Always unregister callback before signaling completion

```

### Q3: Three completion channels and receiver methods

| Channel | Receiver Method | Meaning | Pipeline Adaptor |
| --- | --- | --- | --- |
| **Value** | `set_value(recv, args...)` | Success | `then(f)` |
| **Error** | `set_error(recv, err)` | Failure | `upon_error(f)` |
| **Stopped** | `set_stopped(recv)` | Cancellation | `upon_stopped(f)` |

```cpp

// Exactly ONE completion signal must be called:

// Good:
void start() noexcept {
    if (cancelled)
        stdexec::set_stopped(std::move(recv));   // stopped channel
    else if (error)
        stdexec::set_error(std::move(recv), ep); // error channel
    else
        stdexec::set_value(std::move(recv), 42); // value channel
}

// BAD: calling multiple signals = UB
void bad_start() noexcept {
    stdexec::set_value(std::move(recv), 42);
    stdexec::set_stopped(std::move(recv));  // UB: already completed!
}

```

```cpp

Channel flow through a pipeline:

just(42) | then(f) | upon_error(g) | upon_stopped(h)
  │          │            │                │
  │ value    │ value      │ value          │ value -> consumer
  │          │ error  ----┘ value           │ value -> consumer
  │          │ stopped    │ stopped  -------┘ value -> consumer

Errors pass through then() untouched.
Stopped passes through then() and upon_error() untouched.

```

---

## Notes

- Stop tokens are non-owning; the `stop_source` that controls them lives in the parent scope.
- Polling (`stop_requested()`) is cheap — typically an atomic load.
- Stop callbacks run on the thread that calls `request_stop()` — be careful with locks.
- `in_place_stop_token` is the lightweight, non-type-erased stop token used by stdexec.
- `upon_stopped` converts the stopped channel to a value — useful for providing defaults.
