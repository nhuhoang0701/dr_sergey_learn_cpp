# Understand stop tokens and cancellation in sender pipelines

**Category:** std::execution & Senders/Receivers  
**Item:** #531  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

Stop tokens flow through sender pipelines via the receiver's environment. Operations can poll for cancellation or register callbacks.

| API | Purpose |
| --- | --- |
| `get_stop_token(get_env(recv))` | Extract stop token from receiver |
| `token.stop_requested()` | Poll: was cancellation requested? |
| `stop_callback(token, fn)` | Register callback for stop request |
| `set_stopped(recv)` | Signal cancellation to receiver |
| `stopped_as_optional` | Convert stopped channel to `std::nullopt` |
| `stopped_as_error(err)` | Convert stopped channel to an error |

---

## Self-Assessment

### Q1: Thread a stop_token through a sender pipeline

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <stop_token>

// The stop_token is part of the receiver's ENVIRONMENT.
// Each adaptor (then, when_all, etc.) propagates it automatically.

struct my_cancellable_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(int),
        stdexec::set_stopped_t()
    >;
    int work_items;

    template <typename Receiver>
    struct op {
        using operation_state_concept = stdexec::operation_state_t;
        int work_items;
        Receiver recv;

        void start() noexcept {
            // Step 1: Extract stop token from receiver's environment:
            auto env = stdexec::get_env(recv);
            auto token = stdexec::get_stop_token(env);

            // Step 2: Use it for cooperative cancellation:
            int result = 0;
            for (int i = 0; i < work_items; ++i) {
                if (token.stop_requested()) {
                    std::cout << "Cancelled at iteration " << i << '\n';
                    stdexec::set_stopped(std::move(recv));
                    return;
                }
                result += i;
            }
            stdexec::set_value(std::move(recv), result);
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return op<Receiver>{work_items, std::move(recv)};
    }
};

int main() {
    // Pipeline: stop_token flows through each stage automatically
    auto pipeline = my_cancellable_sender{1000}
        | stdexec::then([](int x) {
            // This stage also gets the stop_token in its environment
            std::cout << "Sum: " << x << '\n';
            return x;
        })
        | stdexec::upon_stopped([]() -> int {
            std::cout << "Was cancelled\n";
            return -1;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';
}
// Output: Sum: 499500  Result: 499500

```

### Q2: Bulk operation with cooperative cancellation

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <vector>
#include <atomic>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    std::vector<double> data(10'000'000, 1.0);

    // bulk() runs a function for each index in a range.
    // Each invocation can check the stop_token:
    auto pipeline =
        stdexec::schedule(sched)
        | stdexec::bulk(data.size(),
            [&data](std::size_t i) {
                // In a real scenario, check stop_token periodically:
                // if (token.stop_requested()) return;
                data[i] = data[i] * 2.0 + 1.0;
            })
        | stdexec::then([&data]() {
            // Verify some results:
            std::cout << "data[0] = " << data[0] << '\n';    // 3.0
            std::cout << "data[999] = " << data[999] << '\n'; // 3.0
        });

    stdexec::sync_wait(std::move(pipeline));
}
// Output:
// data[0] = 3
// data[999] = 3

```

For cancellation in bulk:

```cpp

// Pattern: check stop_token every N iterations
auto pipeline = stdexec::schedule(sched)
    | stdexec::let_value([&data]() {
        return stdexec::just()
            | stdexec::then([&data]() {
                for (std::size_t i = 0; i < data.size(); ++i) {
                    // Inside a sender, access the stop_token from the receiver:
                    // if (i % 10000 == 0 && stop_requested) break;
                    data[i] *= 2.0;
                }
            });
    });

```

### Q3: `stopped_as_optional` — convert cancellation to `std::nullopt`

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>

int main() {
    // A sender that might be stopped:
    auto maybe_cancelled = stdexec::just_stopped()  // simulates cancellation
        | stdexec::upon_stopped([]() { return 42; });
    // upon_stopped converts stopped -> value

    // Alternative: stopped_as_optional
    // Converts: set_value(x) -> optional(x)
    //           set_stopped() -> nullopt
    auto pipeline = stdexec::just_stopped()
        | stdexec::stopped_as_optional();

    auto result = stdexec::sync_wait(std::move(pipeline)).value();
    // result is std::tuple<std::optional<void>> or similar

    // Practical usage:
    auto compute_or_cancel = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; })
        | stdexec::stopped_as_optional();

    // If pipeline completes normally: optional(84)
    // If pipeline is cancelled: nullopt
}

// stopped_as_optional is useful when cancellation is expected
// and you want to handle it as a "no result" rather than an error.
//
// Comparison:
//   upon_stopped([] { return default; })  -- provides fallback value
//   stopped_as_optional()                 -- wraps in optional
//   stopped_as_error(err)                 -- converts to error channel

```

---

## Notes

- Stop tokens propagate automatically through all P2300 adaptors (`then`, `when_all`, `bulk`, etc.).
- `stopped_as_optional` is useful in `when_all` where one branch might cancel.
- Stop callbacks run on the thread calling `request_stop()` — keep them fast.
- `in_place_stop_token` (stdexec) is the default; `std::stop_token` (C++20) can also be used.
- Never ignore the stopped channel — unhandled cancellation propagates outward.
