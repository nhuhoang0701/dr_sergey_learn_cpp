# Compare async patterns: callbacks vs futures vs coroutines vs senders/receivers

**Category:** Concurrency and Parallelism  
**Standard:** C++20/26  
**Reference:** <https://wg21.link/P2300>  

---

## Topic Overview

C++ has evolved through multiple async programming models. Each has distinct tradeoffs.

### Comparison Matrix

| Pattern | Allocation | Composability | Cancellation | Error Handling |
| --- | --- | --- | --- | --- |
| Callbacks | None | Poor (nesting) | Manual | Manual |
| std::future | Shared state (heap) | Poor (no chaining) | None | exception_ptr |
| Coroutines | Frame (heap, elidable) | Good (co_await) | Via token | co_await + try/catch |
| Senders/Receivers | Customizable | Excellent (pipe) | Built-in | set_error channel |

### Callbacks (Oldest)

```cpp

void read_file(const char* path,
               std::function<void(std::string)> on_success,
               std::function<void(int)> on_error) {
    // Pros: zero overhead, simple
    // Cons: callback hell, hard to compose, error handling scattered
}

```

### std::future (C++11)

```cpp

#include <future>
auto f = std::async(std::launch::async, []{ return compute(); });
int result = f.get();  // Blocks until ready
// Pros: simple syntax
// Cons: no chaining, always allocates shared state, no cancellation

```

### Coroutines (C++20)

```cpp

Task<int> compute_async() {
    auto data = co_await read_file_async("input.txt");
    auto result = co_await process_async(data);
    co_return result;
}
// Pros: natural sequential syntax, composable
// Cons: heap allocation for frame, tooling immature

```

### Senders/Receivers (C++26)

```cpp

auto work = ex::just("input.txt")
    | ex::then(read_file)
    | ex::then(process)
    | ex::upon_error([](auto err) { log(err); return default_value(); });

ex::sync_wait(std::move(work));
// Pros: lazy, composable, cancellation, custom execution context
// Cons: complex mental model, verbose

```

---

## Self-Assessment

### Q1: When to use each pattern

- Callbacks: C interop, simple one-shot async operations
- Futures: Simple "fire and forget" parallel computation
- Coroutines: Complex async control flow, generators, async I/O
- Senders: Library-level async composition, custom schedulers, structured concurrency

### Q2: Why are senders better than futures for composition

Senders are lazy (don't start until connected), support cancellation (stop tokens), and are type-safe (the scheduler is part of the type). Futures are eager (start immediately) and can't be cancelled or composed without blocking.

### Q3: Can coroutines and senders work together

Yes. `co_await` can await a sender, and senders can wrap coroutines. P2300 defines `as_awaitable` to bridge between the two models.

---

## Notes

- The industry is converging on coroutines (application code) + senders (library plumbing).
- `std::future` is considered a design mistake — avoid in new code.
- Senders/receivers (P2300) targeting C++26 as `std::execution`.
- Each pattern can be layered: senders internally using coroutines using callbacks.
