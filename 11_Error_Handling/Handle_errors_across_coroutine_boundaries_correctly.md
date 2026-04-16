# Handle errors across coroutine boundaries correctly

**Category:** Error Handling  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

Coroutines have unique error handling challenges: exceptions must be captured in the promise, propagated on `co_await`, and lifetime issues can cause errors to be silently lost.

### Exception Flow in Coroutines

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

### Error Handling Patterns

```cpp

// Pattern 1: co_await propagates exceptions
Task<int> pipeline() {
    auto data = co_await fetch_data();    // If fetch throws → propagated
    auto result = co_await process(data);  // If process throws → propagated
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

---

## Self-Assessment

### Q1: What happens if unhandled_exception is not implemented

If the promise type doesn't have `unhandled_exception()`, the program is ill-formed. If it exists but doesn't capture the exception (empty body), the exception is silently lost — a dangerous bug.

### Q2: Can you use try/catch inside a coroutine

Yes. `try`/`catch` works normally inside coroutines. The function-body `try` block catches exceptions from `co_await` expressions and from the coroutine's own code. However, you cannot have a function-try-block for a coroutine (the entire body can't be wrapped).

### Q3: What is the danger of exceptions + coroutine lifetimes

If a coroutine is destroyed without being resumed after an exception is stored, the exception is never rethrown. The error is silently lost. Always ensure destroyed coroutines have their exceptions checked.

---

## Notes

- Always implement `unhandled_exception()` to capture `std::current_exception()`.
- Prefer `expected`-based coroutines for explicit error handling.
- Check for stored exceptions when destroying incomplete coroutine tasks.
- Structured concurrency (C++26) will help ensure coroutine errors are never lost.
