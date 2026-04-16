# Compare std::execution with coroutines and know when to combine them

**Category:** std::execution & Senders/Receivers  
**Item:** #532  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/execution>  

---

## Topic Overview

Coroutines (C++20) and senders/receivers (P2300/C++26) are complementary async models. Coroutines give sequential-looking code; senders give composable pipelines.

| Aspect | Coroutines | Senders/Receivers |
| --- | --- | --- |
| Syntax | `co_await`, `co_return` | Pipe operator `\|` |
| Abstraction | Suspend/resume points | Lazy value channels |
| Allocation | Coroutine frame (heap by default) | Operation states (often stack) |
| Cancellation | Manual (stop_token wiring) | Built-in (stopped channel) |
| Composition | Sequential by default | Parallel fan-out (when_all) |
| Structured concurrency | Library-provided | First-class (async_scope) |
| Best for | Sequential async logic | Parallel pipelines, scheduler control |

```cpp

Coroutine:                    Sender pipeline:
task<int> foo() {             auto s = just(42)
  auto x = co_await bar();      | then([](int x) {
  auto y = co_await baz(x);         return x * 2;
  co_return x + y;               })
}                               | then(to_string);

```

---

## Self-Assessment

### Q1: Make a sender awaitable in a coroutine with `co_await`

```cpp

// Using stdexec (NVIDIA P2300 reference implementation)
#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <exec/task.hpp>   // provides exec::task coroutine type
#include <iostream>

// exec::task is a coroutine type that can co_await senders:
exec::task<int> compute_async() {
    // co_await a sender → suspends until the sender completes,
    // then resumes with the value:
    int x = co_await stdexec::just(21);
    std::cout << "Got: " << x << '\n';

    // Chain senders inside a coroutine:
    auto doubled = co_await (
        stdexec::just(x)
        | stdexec::then([](int v) { return v * 2; })
    );
    std::cout << "Doubled: " << doubled << '\n';

    co_return doubled;  // 42
}

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // The coroutine itself IS a sender — can be used in pipelines:
    auto pipeline =
        stdexec::starts_on(sched, compute_async())
        | stdexec::then([](int result) {
            std::cout << "Pipeline got: " << result << '\n';
            return result;
        });

    auto [val] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Final: " << val << '\n';  // 42
}
// Output:
// Got: 21
// Doubled: 42
// Pipeline got: 42
// Final: 42

```

**How it works:**

- `exec::task<T>` is both a coroutine type and a sender.
- `co_await sender` connects the sender, starts it, suspends the coroutine, and resumes with the result.
- The coroutine can be composed into sender pipelines via `|`.

### Q2: When to prefer senders vs coroutines

Use **senders** when:

- You need **parallel fan-out**: `when_all(s1, s2, s3)` launches concurrently.
- You want **scheduler control**: `starts_on(gpu_sched, work)` runs on GPU.
- You need **structured concurrency**: `async_scope` ensures child tasks complete.
- No dynamic allocation is needed (operation states can live on the stack).

Use **coroutines** when:

- The logic is **sequential** and would be hard to express as a pipeline.
- You want **readable control flow** with if/else, loops, try/catch.
- You're doing I/O with many suspend points (file reads, network calls).

Use **both** when:

- Sequential coroutine logic needs to co_await parallel sender work.
- A sender pipeline needs a complex step best written as a coroutine.

```cpp

// Combining both: coroutine with parallel sender fan-out
exec::task<double> process_data() {
    // Sequential part (coroutine is natural):
    auto config = co_await load_config();  // sender

    // Parallel part (senders shine):
    auto [a, b, c] = co_await stdexec::when_all(
        fetch_from_db(config.db_url),      // sender
        fetch_from_api(config.api_url),    // sender
        compute_locally(config.data)       // sender
    );
    // All three ran concurrently!

    // Sequential again:
    co_return merge_results(a, b, c);
}

```

### Q3: The `as_awaitable` adaptor

`execution::as_awaitable(sender, promise)` bridges any sender into a coroutine:

```cpp

#include <stdexec/execution.hpp>
#include <exec/task.hpp>
#include <iostream>

// as_awaitable converts a sender into an awaitable:
// This is what exec::task does internally when you co_await a sender.

// Conceptual implementation:
template <typename Sender, typename Promise>
auto as_awaitable(Sender&& sndr, Promise& promise) {
    // 1. Deduces the value type from the sender's completion signatures
    // 2. Creates an awaiter that:
    //    - await_ready(): returns false (always suspends)
    //    - await_suspend(handle): connects sender to a receiver that
    //      resumes the coroutine on completion
    //    - await_resume(): returns the value from set_value()
    //      or rethrows from set_error()
    //      or throws operation_cancelled from set_stopped()
}

exec::task<void> example() {
    // When you write:
    int x = co_await stdexec::just(42);

    // The compiler calls:
    // auto awaiter = tag_invoke(as_awaitable, just(42), promise);
    // if (!awaiter.await_ready()) {
    //     awaiter.await_suspend(coroutine_handle);
    //     // ... suspended ...
    // }
    // int x = awaiter.await_resume();

    std::cout << x << '\n';  // 42
}

```

```cpp

Bridge diagram:

  Coroutine world          Bridge              Sender world
┌─────────────────┐                        ┌──────────────────┐
│ co_await sender  │──→ as_awaitable() ──→  │ connect(s, recv)  │
│                  │                        │ start(op_state)   │
│ ...suspended...  │                        │ ...running...     │
│                  │←── resume handle ←──── │ set_value(42)     │
│ int x = 42      │                        └──────────────────┘
└─────────────────┘

```

---

## Notes

- `exec::task<T>` from stdexec is the recommended coroutine type for P2300 interop.
- Coroutines that co_await senders get **automatic cancellation** via stop tokens.
- Senders avoid coroutine frame heap allocations — better for latency-sensitive code.
- The `as_awaitable` customization point is ADL-discovered via `tag_invoke`.
- In C++26, `std::execution::task` will be the standard coroutine-sender bridge.
