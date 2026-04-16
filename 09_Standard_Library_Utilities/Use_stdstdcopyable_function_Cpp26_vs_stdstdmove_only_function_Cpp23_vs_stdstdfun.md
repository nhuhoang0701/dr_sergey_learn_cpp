# Use std::copyable_function (C++26) vs std::move_only_function (C++23) vs std::function

**Category:** Standard Library Utilities  
**Standard:** C++23/26  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/move_only_function>  

---

## Topic Overview

C++ has three type-erased callable wrappers, each with different ownership semantics.

### Comparison Table

| Feature | `std::function` | `move_only_function` | `copyable_function` |
| --- | --- | --- | --- |
| Standard | C++11 | C++23 | C++26 |
| Copyable | Yes (required) | No | Yes |
| Move-only callables | No | Yes | No |
| const-correctness | Bug (`const` calls non-const) | Correct | Correct |
| noexcept in signature | No | Yes | Yes |

### The const-correctness Problem

```cpp

#include <functional>

// std::function has a design bug:
void bug_example() {
    int counter = 0;
    const std::function<void()> f = [counter]() mutable { ++counter; };
    f();  // Compiles! const function calling mutable lambda
    // This violates const-correctness
}

// C++23 move_only_function fixes this:
// std::move_only_function<void() const> f = ...;
// f() on a const reference only works if the callable is const-invocable

```

### Usage Patterns

```cpp

#include <functional>

// std::function: copyable callbacks (event handlers, shared callbacks)
std::vector<std::function<void(int)>> handlers;
handlers.push_back([](int x) { std::cout << x; });

// move_only_function: move-only callables (unique_ptr captures)
std::move_only_function<void()> task = [p = std::make_unique<int>(42)]() {
    std::cout << *p;
};
// Can't copy task — but can store in a vector and move it

// copyable_function (C++26): like function but with correct const behavior
std::copyable_function<void() const> safe_cb = []() { std::cout << "safe"; };

```

---

## Self-Assessment

### Q1: Why does std::function require copyability

`std::function` was designed before move semantics were widespread. Its API guarantees copyability, which means the wrapped callable must be copyable. This prevents wrapping lambdas that capture `unique_ptr` or other move-only types.

### Q2: When should you use each

- `std::function`: Legacy code, APIs that need copyable callbacks
- `std::move_only_function`: C++23 code with move-only callables, task queues
- `std::copyable_function`: C++26 replacement for `std::function` with correct semantics

### Q3: What is `std::function_ref` (C++26)

A non-owning reference to a callable — like `string_view` for functions. Zero allocation, no ownership. Use when you need to pass a callback to a function but don't need to store it.

---

## Notes

- `std::move_only_function` supports `noexcept` in the signature: `move_only_function<void() noexcept>`.
- `std::function_ref` (C++26) is the lightest option — no heap allocation, no ownership.
- All three support `const`, `noexcept`, and ref-qualification in the signature (except `std::function`).
- Prefer `move_only_function` for task queues and thread pools.
