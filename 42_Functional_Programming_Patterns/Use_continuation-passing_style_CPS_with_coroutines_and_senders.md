# Use continuation-passing style (CPS) with coroutines and senders

**Category:** Functional Programming Patterns  
**Standard:** C++20/26  
**Reference:** <https://wg21.link/P2300>  

---

## Topic Overview

Continuation-passing style (CPS) is a pattern where instead of returning a result, a function takes a "continuation" (callback) that receives the result. Coroutines and the sender/receiver model (P2300) are structured forms of CPS.

### Manual CPS (Callback Style)

```cpp

#include <functional>
#include <string>
#include <iostream>

// CPS: instead of returning, pass result to continuation
void read_file_cps(const std::string& path,
                   std::function<void(std::string)> on_success,
                   std::function<void(std::string)> on_error) {
    // Simulate async IO
    if (path.empty()) {
        on_error("empty path");
    } else {
        on_success("file contents of " + path);
    }
}

// Nesting becomes "callback hell":
void process() {
    read_file_cps("config.json",
        [](std::string config) {
            read_file_cps(config, // pretend config contains a path
                [](std::string data) {
                    std::cout << "Got: " << data << "\n";
                },
                [](std::string err) { std::cerr << err << "\n"; });
        },
        [](std::string err) { std::cerr << err << "\n"; });
}

```

### Coroutines: Structured CPS

```cpp

#include <coroutine>
#include <string>
#include <iostream>

// A coroutine IS CPS — co_await is "pass the continuation"
// The compiler transforms:
//   auto result = co_await async_read("path");
//   process(result);
// Into:
//   async_read("path", [](auto result) { process(result); });

// But with coroutines, it LOOKS like sequential code:
Task<void> process_structured() {
    auto config = co_await async_read("config.json");
    auto data = co_await async_read(config);
    std::cout << "Got: " << data << "\n";
    // No nesting! Same logic, linear structure
}

```

### Senders/Receivers: Composable CPS

```cpp

// P2300 std::execution — structured, composable CPS:
#include <execution>  // Future standard

auto work = ex::just("config.json")
    | ex::then([](auto path) { return read_file(path); })
    | ex::then([](auto config) { return parse_config(config); })
    | ex::upon_error([](auto err) { log_error(err); return default_config(); });

// Each `then` attaches a continuation
// The pipeline is a "sender" — computation doesn't start until connected to a receiver
// ex::sync_wait(work); starts execution

```

---

## Self-Assessment

### Q1: What's the relationship between CPS and coroutines

Coroutines are syntactic sugar for CPS. When the compiler encounters `co_await expr`, it saves the current function state (continuation) and passes it to the awaitable. The awaitable calls the continuation when the result is ready. The programmer writes sequential code; the compiler generates CPS.

### Q2: How do senders improve over raw callbacks

Senders are lazy (computation doesn't start until connected), composable (pipe operators chain operations), and structured (lifetime of work is clear). Raw callbacks are eager, hard to compose, and error-prone (callback hell, missed error handling, dangling references).

### Q3: Show CPS transform for a simple function

```cpp

// Direct style:
int add(int a, int b) { return a + b; }
int result = add(1, 2);
process(result);

// CPS transform:
void add_cps(int a, int b, std::function<void(int)> k) {
    k(a + b);  // Pass result to continuation k
}
add_cps(1, 2, [](int result) { process(result); });

```

---

## Notes

- CPS is the theoretical foundation for async/await in C++, JavaScript, Rust, and C#.
- Senders/receivers (P2300) will be the standard C++ async model (targeting C++26).
- CPS enables tail-call optimization — but C++ doesn't guarantee tail-call elimination.
- Coroutines make CPS ergonomic while maintaining the performance benefits.
