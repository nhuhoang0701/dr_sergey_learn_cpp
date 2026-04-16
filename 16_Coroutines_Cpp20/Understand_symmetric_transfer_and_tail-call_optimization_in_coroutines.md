# Understand symmetric transfer and tail-call optimization in coroutines

**Category:** Coroutines (C++20)  
**Item:** #399  
**Reference:** <https://en.cppreference.com/w/cpp/language/coroutines>  

---

## Topic Overview

**Symmetric transfer** is a mechanism where `await_suspend` returns a `coroutine_handle<>`, and the compiler resumes it via a **tail-call** rather than a nested call. This prevents stack growth in coroutine chains.

### The Problem Without Symmetric Transfer

```cpp

main() calls A.resume()
  A runs, calls B.resume() in await_suspend
    B runs, calls C.resume() in await_suspend
      C runs, calls D.resume() in await_suspend
        ... stack grows with each resume!

```

### The Solution With Symmetric Transfer

```cpp

main() calls A.resume()
  A suspends, await_suspend returns B's handle
  → compiler does tail-call to B (A's frame popped)
  B suspends, await_suspend returns C's handle
  → compiler does tail-call to C (B's frame popped)
  ... stack NEVER grows beyond one frame

```

### await_suspend Return Types

| Return type | Stack effect | Use case |
| --- | --- | --- |
| `void` | Frame stays on stack | Simple suspend |
| `bool` | Frame stays if false | Conditional suspend |
| `coroutine_handle<>` | **Tail-call** — frame popped | Symmetric transfer |

---

## Self-Assessment

### Q1: Explain how symmetric transfer avoids stack growth when a coroutine resumes another coroutine

**Without symmetric transfer (stack grows):**

```cpp

void await_suspend(std::coroutine_handle<> h) {
    other_handle.resume();  // NESTED CALL — h's frame stays on stack
    // When other_handle suspends, we return here
    // Stack: main -> A -> B -> C -> ...
}

```

**With symmetric transfer (stack stays flat):**

```cpp

std::coroutine_handle<> await_suspend(std::coroutine_handle<> h) {
    return other_handle;  // TAIL-CALL — h's frame is popped
    // Compiler generates: "pop my frame, then jump to other_handle"
    // Stack: main -> current_coroutine (only one at a time)
}

```

**Stack comparison for 1000 chained coroutines:**

```cpp

Without symmetric transfer:
  Stack depth: O(N) — stack overflow at ~1000-10000 coroutines
  [main][A][B][C][D]...[1000]  ← 💥 stack overflow

With symmetric transfer:
  Stack depth: O(1) — always constant
  [main][current]  ← only one active frame at a time

```

The key insight: when `await_suspend` returns a `coroutine_handle<>`, the compiler:

1. Pops the current coroutine's local frame from the call stack
2. Jumps to the returned handle (like a `goto`, not a `call`)
3. Result: constant stack usage regardless of chain depth

### Q2: Implement a coroutine that uses symmetric transfer via `await_suspend` returning `coroutine_handle`

```cpp

#include <coroutine>
#include <iostream>
#include <optional>
#include <utility>

template<typename T>
class SymTask {
public:
    struct promise_type {
        std::optional<T> value;
        std::coroutine_handle<> continuation;

        SymTask get_return_object() {
            return SymTask{std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() noexcept { return {}; }

        // SYMMETRIC TRANSFER at final_suspend
        auto final_suspend() noexcept {
            struct TransferAwaiter {
                std::coroutine_handle<> cont;

                bool await_ready() noexcept { return false; }

                // Returns a handle → symmetric transfer (tail-call)
                std::coroutine_handle<> await_suspend(
                    std::coroutine_handle<>) noexcept {
                    return cont ? cont : std::noop_coroutine();
                }

                void await_resume() noexcept {}
            };
            return TransferAwaiter{continuation};
        }

        void return_value(T v) { value = std::move(v); }
        void unhandled_exception() { std::terminate(); }
    };

private:
    std::coroutine_handle<promise_type> handle_;
    explicit SymTask(std::coroutine_handle<promise_type> h) : handle_(h) {}

public:
    SymTask(SymTask&& o) noexcept : handle_(std::exchange(o.handle_, {})) {}
    ~SymTask() { if (handle_) handle_.destroy(); }

    // co_await support with symmetric transfer
    bool await_ready() const noexcept { return false; }

    std::coroutine_handle<> await_suspend(std::coroutine_handle<> caller) noexcept {
        handle_.promise().continuation = caller;
        return handle_;  // symmetric transfer: start this task
    }

    T await_resume() { return std::move(*handle_.promise().value); }

    T sync_get() {
        handle_.resume();
        return std::move(*handle_.promise().value);
    }
};

// Chain: inner -> middle -> outer (all use symmetric transfer)
SymTask<int> inner() {
    co_return 10;
}

SymTask<int> middle() {
    int val = co_await inner();  // symmetric transfer to inner
    co_return val * 2;
}

SymTask<int> outer() {
    int val = co_await middle();  // symmetric transfer to middle
    co_return val + 5;
}

int main() {
    auto t = outer();
    std::cout << "Result: " << t.sync_get() << '\n';
}
// Expected output:
// Result: 25
// (inner returns 10, middle returns 20, outer returns 25)
// Stack never grew beyond 1 active coroutine frame

```

### Q3: Show that without symmetric transfer, deeply chained coroutines can cause stack overflow

```cpp

#include <coroutine>
#include <iostream>

// Task WITHOUT symmetric transfer (uses void await_suspend)
struct UnsafeTask {
    struct promise_type {
        int value = 0;
        UnsafeTask get_return_object() {
            return {std::coroutine_handle<promise_type>::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(int v) { value = v; }
        void unhandled_exception() { std::terminate(); }
    };

    std::coroutine_handle<promise_type> handle;
    ~UnsafeTask() { if (handle) handle.destroy(); }

    bool await_ready() { return false; }

    // NO symmetric transfer — resumes inline (stack grows!)
    void await_suspend(std::coroutine_handle<> caller) {
        handle.resume();  // NESTED CALL — stack grows
        caller.resume();  // return to caller after
    }

    int await_resume() { return handle.promise().value; }
};

// This would stack overflow with deep enough recursion:
// UnsafeTask deep_chain(int depth) {
//     if (depth == 0) co_return 0;
//     int v = co_await deep_chain(depth - 1);  // stack grows each level!
//     co_return v + 1;
// }
// deep_chain(100000) → STACK OVERFLOW

// With symmetric transfer (from Q2's SymTask), the same chain is safe:
// SymTask<int> safe_chain(int depth) {
//     if (depth == 0) co_return 0;
//     int v = co_await safe_chain(depth - 1);  // tail-call, O(1) stack
//     co_return v + 1;
// }
// safe_chain(100000) → works fine!

int main() {
    std::cout << "Symmetric transfer prevents stack overflow in deep chains.\n";
    std::cout << "Without it: each co_await adds a stack frame.\n";
    std::cout << "With it: await_suspend returns handle -> tail-call -> O(1) stack.\n";
}

```

**Summary:**

| Aspect | `void await_suspend` | `handle<> await_suspend` |
| --- | --- | --- |
| Stack growth | O(N) per chain depth | O(1) constant |
| Max chain depth | ~1K-10K (stack limit) | Unlimited |
| Code pattern | `other.resume();` in body | `return other;` |
| Compiler behavior | Nested function call | Tail-call optimization |

---

## Notes

- `std::noop_coroutine()` is the "do nothing" handle used when there's no continuation to transfer to.
- Symmetric transfer is essential for production coroutine libraries (cppcoro, folly::coro, libunifex).
- All major compilers (GCC, Clang, MSVC) support symmetric transfer with `-O1` or higher.
- Lewis Baker's blog posts and P0913R0 are the canonical references for this technique.
