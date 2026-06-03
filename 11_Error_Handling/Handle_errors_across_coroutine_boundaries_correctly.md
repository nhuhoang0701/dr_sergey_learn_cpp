# Handle errors across coroutine boundaries correctly

**Category:** Error Handling  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Coroutines have unique error handling challenges: exceptions must be captured in the promise, propagated on `co_await`, and lifetime issues can cause errors to be silently lost. This last point is the sneaky one - with ordinary functions, an unhandled exception terminates the program. With coroutines, an exception can disappear without a trace if the coroutine is destroyed before anyone asks for its result.

### Exception Flow in Coroutines

The reason exceptions need special handling in coroutines is that a coroutine is not a regular call stack. When an exception escapes the coroutine body, it doesn't unwind to the caller - instead it lands in `unhandled_exception()` on the promise type. From there, the promise stores it via `std::current_exception()` and it waits, frozen, until someone calls `result()` and re-throws it. Here's a minimal `Task` type that demonstrates the full flow.

```cpp
#include <coroutine>
#include <exception>
#include <stdexcept>
#include <iostream>

template<typename T>
struct Task {
    struct promise_type {
        T result_;
        std::exception_ptr exception_;

        Task get_return_object() {
            return Task{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }

        void return_value(T val) { result_ = std::move(val); }

        void unhandled_exception() {
            exception_ = std::current_exception();  // Capture!
        }
    };

    std::coroutine_handle<promise_type> handle_;

    T result() {
        if (handle_.promise().exception_)
            std::rethrow_exception(handle_.promise().exception_);
        return handle_.promise().result_;
    }
};

Task<int> might_fail() {
    throw std::runtime_error("oops");
    co_return 42;  // Never reached
}

int main() {
    auto task = might_fail();
    task.handle_.resume();
    try {
        int val = task.result();  // Rethrows the captured exception
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";  // "oops"
    }
}
```

The exception is thrown inside the coroutine body, captured in `unhandled_exception`, and re-thrown only when `result()` is called. If `result()` is never called - say, the `Task` is destroyed early - the exception is silently swallowed.

### Error Handling Patterns

Here are the two main approaches you'll see in practice. Pattern 1 uses regular exceptions with `co_await` propagating them naturally. Pattern 2 avoids exceptions entirely by using `std::expected` as the return type, which makes the error an explicit part of the value.

```cpp
// Pattern 1: co_await propagates exceptions
Task<int> pipeline() {
    auto data = co_await fetch_data();    // If fetch throws -> propagated
    auto result = co_await process(data);  // If process throws -> propagated
    co_return result;
    // try/catch works naturally:
    // try { auto x = co_await risky(); } catch(...) { co_return -1; }
}

// Pattern 2: expected-based coroutines (no exceptions)
Task<std::expected<int, Error>> safe_pipeline() {
    auto data = co_await fetch_data();
    if (!data) co_return std::unexpected(data.error());
    auto result = co_await process(*data);
    co_return result;
}
```

Pattern 2 is often better for libraries or for code that runs at high frequency. Returning `std::expected` makes it impossible to accidentally swallow an error - every caller is forced to inspect the return value.

---

## Self-Assessment

### Q1: What happens if unhandled_exception is not implemented

If the promise type doesn't have `unhandled_exception()`, the program is ill-formed. If it exists but doesn't capture the exception (empty body), the exception is silently lost - a dangerous bug. The reason this trips people up is that an empty `unhandled_exception()` compiles and runs fine most of the time; you only discover the problem when your coroutine silently fails to report an error in production.

### Q2: Can you use try/catch inside a coroutine

Yes. `try`/`catch` works normally inside coroutines. The function-body `try` block catches exceptions from `co_await` expressions and from the coroutine's own code. However, you cannot have a function-try-block for a coroutine (the entire body can't be wrapped the way you can with a regular function or constructor).

### Q3: What is the danger of exceptions + coroutine lifetimes

If a coroutine is destroyed without being resumed after an exception is stored, the exception is never rethrown. The error is silently lost. Always ensure destroyed coroutines have their exceptions checked. This is one of the motivating problems for structured concurrency (coming in C++26), which ensures that coroutine errors are always delivered to someone.

---

## Notes

- Always implement `unhandled_exception()` to capture `std::current_exception()`. An empty body is a silent bug factory.
- Prefer `expected`-based coroutines for explicit error handling when exceptions are too heavy or too easy to miss.
- Check for stored exceptions when destroying incomplete coroutine tasks.
- Structured concurrency (C++26) will help ensure coroutine errors are never lost.
