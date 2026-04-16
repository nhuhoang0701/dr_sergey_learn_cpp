# Understand coroutine interaction with exceptions and stack unwinding

**Category:** Coroutines (C++20)  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

### How Exceptions Work in Coroutines

Coroutines have a unique exception handling model compared to regular functions:

1. When an exception escapes the coroutine body, it's caught by the compiler-generated wrapper.
2. The `promise_type::unhandled_exception()` method is called.
3. The exception can be stored (via `std::current_exception()`) and rethrown later.
4. Stack unwinding destroys local variables in the coroutine frame, but the frame itself stays alive until the handle is destroyed.

### The Compiler-Generated Try/Catch

The compiler transforms a coroutine roughly like this:

```cpp

// What you write:
task<int> compute() {
    auto data = co_await fetch();
    co_return process(data);
}

// What the compiler generates (conceptual):
task<int> compute() {
    auto& promise = get_promise();
    auto return_object = promise.get_return_object();

    try {
        co_await promise.initial_suspend();
        // --- your code ---
        auto data = co_await fetch();
        co_return process(data);
        // --- end your code ---
    } catch (...) {
        promise.unhandled_exception();  // <-- exception caught here
    }
    co_await promise.final_suspend();
}

```

### Implementing `unhandled_exception()`

The standard approach stores the exception for later rethrowing:

```cpp

struct task_promise {
    std::exception_ptr exception_;

    void unhandled_exception() {
        // Store the current exception
        exception_ = std::current_exception();
    }

    // When the caller awaits the result, rethrow if needed
    int get_result() {
        if (exception_)
            std::rethrow_exception(exception_);
        return value_;
    }
};

```

### Exception Safety at Suspension Points

Exceptions can happen at different stages of `co_await`:

```cpp

task<void> example() {
    // Stage 1: evaluating the awaitable expression — exceptions propagate normally
    auto awaitable = get_awaitable();  // may throw

    // Stage 2: await_ready() — may throw (before suspension)
    // Stage 3: await_suspend() — if it throws, the coroutine is resumed immediately
    // Stage 4: await_resume() — may throw (after resumption)

    auto result = co_await std::move(awaitable);
    // If await_resume() throws, exception enters the promise's unhandled_exception
}

```

### Critical: Exceptions in `await_suspend()`

If `await_suspend()` throws, **the coroutine is immediately resumed** and the exception propagates to the caller of `resume()` or to the coroutine body (depending on the form):

```cpp

struct dangerous_awaiter {
    bool await_ready() { return false; }

    void await_suspend(std::coroutine_handle<> h) {
        throw std::runtime_error("suspend failed");
        // The coroutine is resumed, and this exception propagates
        // to the coroutine body's implicit try/catch
    }

    void await_resume() {}
};

```

### Exception Propagation Across Coroutine Boundaries

```cpp

task<int> inner() {
    throw std::runtime_error("inner error");
    co_return 42;  // never reached
}

task<void> outer() {
    try {
        int val = co_await inner();
        // inner()'s exception is rethrown here by await_resume()
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
        // Output: Caught: inner error
    }
}

```

### `noexcept` Coroutines

You **cannot** declare a coroutine as `noexcept` in most implementations. If you try, and an exception would be thrown, `std::terminate()` is called:

```cpp

// Careful: if unhandled_exception() calls std::terminate(),
// an exception in the body terminates the program
struct terminate_on_exception_promise {
    void unhandled_exception() {
        std::terminate();  // hard crash
    }
};

// Better: log and store
struct safe_promise {
    void unhandled_exception() {
        std::cerr << "Coroutine exception caught\n";
        exception_ = std::current_exception();
    }
};

```

---

## Self-Assessment

### Q1: What happens when an exception escapes a coroutine body

1. The compiler-generated code catches the exception.
2. `promise.unhandled_exception()` is called — typically stores the exception via `std::current_exception()`.
3. Execution continues to `promise.final_suspend()`.
4. When the caller `co_await`s the result and calls `await_resume()`, the stored exception is rethrown.
5. Local variables in the coroutine frame are destroyed (stack unwinding of the coroutine frame).

### Q2: Demonstrate exception propagation from an inner coroutine to an outer one

```cpp

#include <iostream>
#include <stdexcept>

task<std::string> read_file(std::string path) {
    if (path.empty())
        throw std::invalid_argument("empty path");
    co_return "file contents";
}

task<void> process() {
    try {
        auto content = co_await read_file("");  // throws
        std::cout << content << "\n";           // never reached
    } catch (const std::invalid_argument& e) {
        std::cout << "Error: " << e.what() << "\n";
        // Output: Error: empty path
    }
}

```

### Q3: Why is `final_suspend` usually `noexcept`

`final_suspend()` is called after the coroutine body completes (including after `unhandled_exception()`). If `final_suspend()` itself throws, the behavior is undefined. Therefore:

```cpp

struct my_promise {
    // MUST be noexcept — throwing here is UB
    auto final_suspend() noexcept {
        return std::suspend_always{};
    }
};

```

The `noexcept` on `final_suspend` is a standard requirement to avoid double-exception scenarios.

---

## Notes

- Always implement `unhandled_exception()` — if you forget, exceptions silently disappear.
- Store exceptions with `std::current_exception()` and rethrow in `await_resume()`.
- Exceptions in `await_suspend()` have surprising semantics — avoid throwing there.
- `final_suspend()` must be `noexcept` — this is enforced by the standard.
- Coroutine exception handling adds overhead; consider `std::expected` for performance-critical paths.
