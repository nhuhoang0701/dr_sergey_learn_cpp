# Survey coroutine libraries: cppcoro, libunifex, folly::coro, stdexec

**Category:** Coroutines (C++20)  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

The C++20 coroutine specification provides **low-level machinery** (compiler transforms, `coroutine_handle`, promise types) but no ready-to-use coroutine types like `task<T>` or `generator<T>`. Several libraries fill this gap.

### Library Comparison

| Library | Maintainer | Status | Key Types | Structured Concurrency |
| --- | --- | --- | --- | --- |
| **cppcoro** | Lewis Baker | Archived (reference impl) | `task<T>`, `generator<T>`, `async_generator<T>` | Basic |
| **libunifex** | Facebook/Meta | Active (P2300 reference) | `task<T>`, schedulers, senders | Yes |
| **folly::coro** | Facebook/Meta | Production | `Task<T>`, `AsyncGenerator<T>`, `co_invoke` | Yes |
| **stdexec** | NVIDIA | Active (P2300 reference) | Senders/receivers, `sync_wait` | Yes |
| **std::generator** | Standard | C++23 | `std::generator<T>` | N/A |

### cppcoro

The original C++20 coroutine library by Lewis Baker. Now archived but widely referenced.

```cpp

#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <cppcoro/when_all.hpp>

cppcoro::task<int> compute() {
    co_return 42;
}

cppcoro::task<std::string> fetch() {
    co_return "hello";
}

int main() {
    // sync_wait drives a coroutine from synchronous code
    auto [val, str] = cppcoro::sync_wait(
        cppcoro::when_all(compute(), fetch())
    );
    // val == 42, str == "hello"
}

```

**Key features:** `task<T>`, `shared_task<T>`, `generator<T>`, `async_generator<T>`, `async_mutex`, `when_all`, `sync_wait`.

### libunifex

Facebook's library that implements the P2300 sender/receiver model with coroutine integration.

```cpp

#include <unifex/task.hpp>
#include <unifex/sync_wait.hpp>
#include <unifex/just.hpp>
#include <unifex/then.hpp>
#include <unifex/thread_unsafe_event_loop.hpp>

unifex::task<int> async_add(int a, int b) {
    co_return a + b;
}

int main() {
    unifex::thread_unsafe_event_loop loop;
    auto result = unifex::sync_wait(
        unifex::on(loop.get_scheduler(), async_add(3, 4))
    );
    // result == 7
}

```

**Key features:** P2300 sender/receiver, schedulers, `let_value`, `when_all`, cancellation via stop tokens.

### folly::coro

Production-grade library used within Meta. Feature-rich with extensive async primitive support.

```cpp

#include <folly/experimental/coro/Task.h>
#include <folly/experimental/coro/BlockingWait.h>
#include <folly/experimental/coro/Collect.h>

folly::coro::Task<int> compute_async() {
    co_return 42;
}

folly::coro::Task<void> example() {
    auto result = co_await compute_async();
    // result == 42

    // Concurrent fan-out
    auto [a, b] = co_await folly::coro::collectAll(
        compute_async(), compute_async());
}

int main() {
    folly::coro::blockingWait(example());
}

```

**Key features:** `Task<T>`, `AsyncGenerator<T>`, `collectAll`, `collectAny`, timeouts, `co_invoke`, retry policies, `AsyncScope`.

### stdexec (P2300 Reference Implementation)

NVIDIA's reference implementation of `std::execution` (P2300).

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    auto work = stdexec::starts_on(sched,
        stdexec::just(40)
        | stdexec::then([](int x) { return x + 2; })
    );

    auto [result] = stdexec::sync_wait(std::move(work)).value();
    // result == 42
}

```

### Choosing a Library

| Need | Recommendation |
| --- | --- |
| Learning / prototyping | cppcoro (simple API) |
| P2300 sender/receiver model | stdexec or libunifex |
| Production async framework | folly::coro |
| Standard-only (no deps) | `std::generator` (C++23) + hand-rolled `task<T>` |
| GPU / heterogeneous | stdexec (NVIDIA backing) |

---

## Self-Assessment

### Q1: Write a minimal coroutine using cppcoro-style `task<T>`

```cpp

#include <cppcoro/task.hpp>
#include <cppcoro/sync_wait.hpp>
#include <iostream>

cppcoro::task<int> fibonacci(int n) {
    if (n <= 1) co_return n;
    auto a = co_await fibonacci(n - 1);
    auto b = co_await fibonacci(n - 2);
    co_return a + b;
}

int main() {
    int result = cppcoro::sync_wait(fibonacci(10));
    std::cout << "fib(10) = " << result << "\n";  // 55
}

```

### Q2: Show concurrent fan-out with folly::coro

```cpp

folly::coro::Task<int> fetch_price(std::string symbol) {
    co_await folly::coro::sleep(std::chrono::milliseconds(100));
    co_return symbol.size() * 10;  // dummy
}

folly::coro::Task<void> portfolio() {
    auto [a, b, c] = co_await folly::coro::collectAll(
        fetch_price("AAPL"),
        fetch_price("GOOG"),
        fetch_price("MSFT")
    );
    std::cout << "Total: " << a + b + c << "\n";
}

```

### Q3: When would you choose stdexec over folly::coro

Choose **stdexec** when:

- You want alignment with the upcoming **C++ standard** (`std::execution`).
- You need **heterogeneous execution** (CPU + GPU) — stdexec supports GPU schedulers.
- You prefer the **sender/receiver** composition model over coroutine-first design.
- You want to avoid the folly dependency ecosystem.

Choose **folly::coro** when:

- You need a **battle-tested production** library.
- You want rich async primitives (retries, timeouts, scoped tasks) out of the box.
- Your team is already using the folly ecosystem.

---

## Notes

- C++23 added `std::generator<T>` to the standard — the first standard coroutine type.
- P2300 (`std::execution`) is targeting C++26 and will provide standard sender/receiver types.
- Most libraries define their own `task<T>` — they are not interchangeable without adapters.
- Coroutine library choice is a major architectural decision — pick early and stick with it.
