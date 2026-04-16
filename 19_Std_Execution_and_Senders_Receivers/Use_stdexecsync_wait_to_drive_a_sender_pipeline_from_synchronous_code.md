# Use stdexec::sync_wait to drive a sender pipeline from synchronous code

**Category:** std::execution & Senders/Receivers  
**Item:** #706  
**Standard:** C++17  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

`sync_wait` is the **bridge** between the lazy, async sender world and normal synchronous code. It blocks the calling thread until the entire sender pipeline completes.

| Behavior | Detail |
| --- | --- |
| Blocks? | Yes — blocks current thread |
| Return type | `std::optional<std::tuple<Values...>>` |
| On value | Returns `optional(tuple(values...))` |
| On error | Rethrows the exception |
| On stopped | Returns `std::nullopt` |
| Execution context | Internal `run_loop` (unless sender specifies another) |

```cpp

sync_wait flow:

  Synchronous code          Async sender world
  ┌──────────────┐      ┌──────────────────┐
  │ auto result =  │ ───▶│ connect(sender,  │
  │  sync_wait(s)  │      │   receiver)     │
  │              │      │ start(op_state) │
  │ // blocked   │      │ ... work ...    │
  │              │ ◀───│ set_value(42)   │
  │ // unblocked │      └──────────────────┘
  │ result = 42  │
  └──────────────┘

```

---

## Self-Assessment

### Q1: `sync_wait` blocks until the pipeline completes

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <chrono>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto pipeline = stdexec::schedule(sched)
        | stdexec::then([]() {
            std::cout << "Working on thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        })
        | stdexec::then([](int x) {
            return x * 2;
        });

    std::cout << "Before sync_wait (main thread: "
              << std::this_thread::get_id() << ")\n";

    // sync_wait blocks here until the pipeline completes:
    auto result = stdexec::sync_wait(std::move(pipeline));

    std::cout << "After sync_wait\n";

    // Unpack the result:
    auto [value] = result.value();
    std::cout << "Result: " << value << '\n';  // 84
}
// Output:
// Before sync_wait (main thread: 140000001)
// Working on thread: 140000050
// After sync_wait
// Result: 84

```

### Q2: Return type is `optional<tuple<...>>` — `nullopt` means stopped

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <optional>
#include <tuple>
#include <string>

int main() {
    // Case 1: Value channel -> optional(tuple(values...))
    {
        auto s = stdexec::just(42, std::string("hello"));
        auto result = stdexec::sync_wait(std::move(s));
        // result type: std::optional<std::tuple<int, std::string>>

        if (result) {
            auto [num, str] = *result;
            std::cout << num << ", " << str << '\n';  // 42, hello
        }
    }

    // Case 2: Error channel -> throws exception
    {
        auto s = stdexec::just_error(
            std::make_exception_ptr(std::runtime_error("oops")));
        try {
            stdexec::sync_wait(std::move(s));
        } catch (const std::runtime_error& e) {
            std::cout << "Caught: " << e.what() << '\n';  // Caught: oops
        }
    }

    // Case 3: Stopped channel -> nullopt
    {
        auto s = stdexec::just_stopped();
        auto result = stdexec::sync_wait(std::move(s));
        // result type: std::optional<std::tuple<>>

        if (!result) {
            std::cout << "Stopped (nullopt)\n";
        }
    }

    // Case 4: Void sender -> optional<tuple<>>
    {
        auto s = stdexec::just();
        auto result = stdexec::sync_wait(std::move(s));
        if (result) {
            std::cout << "Void sender completed\n";
        }
    }
}
// Output:
// 42, hello
// Caught: oops
// Stopped (nullopt)
// Void sender completed

```

### Q3: `sync_wait` is the bridge between async and sync

| Aspect | Without sync_wait | With sync_wait |
| --- | --- | --- |
| Pipeline state | Lazy (no work done) | Drives execution |
| Thread | No thread runs it | Internal run_loop or pool |
| Result | Inaccessible | Returned as optional<tuple> |
| Use case | Composition | Entry/exit point |

```cpp

// sync_wait is typically used only at the "edge" of your program:

int main() {
    // Build a complex pipeline (no work happens yet):
    auto pipeline = build_complex_pipeline();

    // sync_wait is the ONLY place where work actually executes:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();

    // DO NOT use sync_wait inside a sender pipeline!
    // BAD: just(42) | then([](int x) {
    //          sync_wait(another_sender);  // DEADLOCK risk!
    //      });
    // Use let_value instead for nested async work.

    return result;
}

```

**Key rules:**

- Use `sync_wait` only at program boundaries (`main`, test functions, thread entry points).
- Never call `sync_wait` from within a sender callback — it can deadlock.
- For nested async work, use `let_value` to return a sender instead.
- `sync_wait` provides its own `run_loop` execution context unless the sender specifies one via `starts_on`.

---

## Notes

- `sync_wait` is the only way to extract a value from a sender pipeline.
- It creates a `run_loop`, connects a receiver to the sender, and runs the loop until completion.
- `sync_wait_with_variant` handles senders with multiple completion signatures.
- In the standard (`<execution>`), it's `std::this_thread::sync_wait`.
- Calling `sync_wait` from a thread pool thread can deadlock if the pipeline needs that same thread.
