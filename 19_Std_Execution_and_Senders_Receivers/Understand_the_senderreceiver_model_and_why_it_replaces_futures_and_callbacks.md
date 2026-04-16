# Understand the sender/receiver model and why it replaces futures and callbacks

**Category:** std::execution & Senders/Receivers  
**Item:** #521  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html>  

---

## Topic Overview

P2300 replaces `std::future` and raw callbacks with a structured model of three primitives:

| Primitive | Role | Lifetime |
| --- | --- | --- |
| **Sender** | Describes async work (lazy) | Until connected |
| **Receiver** | Continuation that handles result | Part of operation state |
| **Operation State** | Live work binding sender+receiver | Until completion signal |

Why replace futures?

- `std::future` requires heap allocation for shared state.
- `std::future` has no built-in chaining (no `.then()` in standard C++).
- `std::future` has no cancellation.
- `std::future` forces eager execution.

---

## Self-Assessment

### Q1: Three async primitives

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // 1. SENDER: describes work (lazy)
    auto sender = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; });
    // Nothing has executed yet!

    // 2. RECEIVER: handles the result
    // (sync_wait creates an internal receiver that stores the result)

    // 3. OPERATION STATE: live work
    // connect(sender, receiver) -> operation_state
    // start(operation_state) -> begins execution
    // On completion: set_value(receiver, 84)

    auto [result] = stdexec::sync_wait(std::move(sender)).value();
    std::cout << result << '\n';  // 84
}

// What sync_wait does internally:
// 1. Creates a receiver (stores result + synchronization)
// 2. auto op = connect(sender, receiver)   // binds them
// 3. start(op)                              // runs the work
// 4. Blocks until receiver is signaled
// 5. Returns the stored result

```

### Q2: Why `std::future::then()` is not composable but sender pipelines are

```cpp

// std::future problems:
#include <future>
#include <iostream>

void future_problems() {
    // Problem 1: No chaining (no .then() in standard C++)
    auto f = std::async(std::launch::async, []() { return 42; });
    int result = f.get();  // BLOCKS here
    // Can't do: f.then([](int x) { return x * 2; }).then(print);

    // Problem 2: Eager execution (work starts immediately)
    auto f2 = std::async(std::launch::async, heavy_work);
    // heavy_work is ALREADY running, can't compose first

    // Problem 3: No error channel (exceptions only via get())
    // Problem 4: No cancellation
    // Problem 5: Heap allocation for shared state
}

// Sender pipeline: fully composable, lazy, zero-allocation
#include <stdexec/execution.hpp>

void sender_solution() {
    // Composable chaining with pipe operator:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) { return x * 2; })     // chain 1
        | stdexec::then([](int x) { return x + 10; })     // chain 2
        | stdexec::upon_error([](auto) { return -1; })     // error handling
        | stdexec::upon_stopped([]() { return 0; });        // cancellation
    // NOTHING has executed! Pure description.

    // Execute when ready:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    // result = 94
}

```

| Feature | `std::future` | Sender Pipeline |
| --- | --- | --- |
| Chaining | No `.then()` in standard | `\| then(f) \| then(g)` |
| Laziness | Eager (runs immediately) | Lazy (runs on sync_wait) |
| Allocation | Heap (shared state) | Stack (operation state) |
| Error handling | Exception via `get()` | Typed error channel |
| Cancellation | None | Stop tokens |
| Parallel fan-out | Manual | `when_all(s1, s2, s3)` |

### Q3: Separating algorithm from execution context

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

// The SAME algorithm works on ANY scheduler:
auto make_pipeline(int input) {
    return stdexec::just(input)
        | stdexec::then([](int x) { return x * x; })
        | stdexec::then([](int x) {
            std::cout << "Result: " << x
                      << " on thread " << std::this_thread::get_id() << '\n';
            return x;
        });
}

int main() {
    exec::static_thread_pool pool(4);
    auto pool_sched = pool.get_scheduler();

    // Same pipeline, different execution contexts:

    // 1. Run inline (sync_wait's run_loop scheduler):
    auto [r1] = stdexec::sync_wait(make_pipeline(5)).value();

    // 2. Run on thread pool:
    auto [r2] = stdexec::sync_wait(
        stdexec::starts_on(pool_sched, make_pipeline(5))
    ).value();

    // Algorithm code is IDENTICAL.
    // Only the scheduler changes where it runs.
    std::cout << r1 << " == " << r2 << '\n';  // 25 == 25
}

```

---

## Notes

- Senders are to async what ranges are to synchronous sequences: lazy, composable, value-semantic.
- The P2300 model is inspired by Facebook's Folly executors and libunifex.
- `std::future` is not deprecated but is considered insufficient for modern async.
- Sender pipelines enable the compiler to see the entire computation graph for optimization.
- C++26 will include `std::execution` in the standard library.
