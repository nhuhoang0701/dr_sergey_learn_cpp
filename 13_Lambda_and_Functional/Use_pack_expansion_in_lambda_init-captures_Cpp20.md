# Use pack expansion in lambda init-captures (C++20)

**Category:** Lambda & Functional  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

### The Problem Before C++20

In C++17 and earlier, you could not directly capture a parameter pack by move into a lambda using init-captures. The workaround was wrapping the pack in a `std::tuple`:

```cpp

// C++17 workaround — clunky
template<typename... Args>
auto make_deferred_call_17(Args&&... args) {
    return [tup = std::make_tuple(std::forward<Args>(args)...)]() mutable {
        std::apply([](auto&... a) { /* use a... */ }, tup);
    };
}

```

### C++20: Pack Expansion in Init-Captures

C++20 allows **pack expansion** directly in lambda init-captures with the `...` syntax:

```cpp

template<typename... Args>
auto make_deferred_call(Args&&... args) {
    // Each element of the pack gets its own init-capture
    return [...captured = std::forward<Args>(args)]() mutable {
        // Use captured... as a pack inside the lambda
        (std::cout << ... << captured) << "\n";
    };
}

```

The syntax `...name = expr` creates one capture per pack element.

### How It Works

```cpp

template<typename F, typename... Args>
auto bind_front_simple(F f, Args&&... args) {
    // ...bound = std::forward<Args>(args) captures each arg by move/copy
    return [f = std::move(f), ...bound = std::forward<Args>(args)]
           (auto&&... remaining) mutable {
        return f(bound..., std::forward<decltype(remaining)>(remaining)...);
    };
}

int add(int a, int b, int c) { return a + b + c; }

int main() {
    auto add_10_20 = bind_front_simple(add, 10, 20);
    std::cout << add_10_20(30) << "\n";  // 60
    return 0;
}

```

When instantiated with `Args = {int, int}`:

- The capture becomes `...bound = {10, 20}` — effectively two captures `bound_0 = 10, bound_1 = 20`.
- Inside the body, `bound...` expands back to the pack.

### Practical Uses

**1. Perfect-forwarding capture for deferred execution:**

```cpp

template<typename F, typename... Args>
auto defer(F&& f, Args&&... args) {
    return [f = std::forward<F>(f),
            ...captured_args = std::forward<Args>(args)]() mutable {
        return std::invoke(std::move(f), std::move(captured_args)...);
    };
}

```

**2. Capturing a pack for a coroutine:**

```cpp

template<typename... Args>
task<void> log_all(Args&&... args) {
    // Capture by move before the first suspension point
    auto logger = [...vals = std::forward<Args>(args)]() {
        (std::cout << ... << vals) << "\n";
    };
    co_await some_async_op();
    logger();
}

```

**3. Building variadic event payloads:**

```cpp

template<typename... Payload>
auto make_event(std::string name, Payload&&... data) {
    return [name = std::move(name),
            ...payload = std::forward<Payload>(data)]() {
        std::cout << "Event: " << name << " with "
                  << sizeof...(payload) << " args\n";
    };
}

```

---

## Self-Assessment

### Q1: Move-capture a parameter pack into a lambda and invoke it

```cpp

#include <iostream>
#include <string>
#include <utility>

template<typename... Args>
auto capture_and_print(Args&&... args) {
    return [...vals = std::forward<Args>(args)]() {
        ((std::cout << vals << " "), ...) ;
        std::cout << "\n";
    };
}

int main() {
    auto fn = capture_and_print(1, 3.14, std::string("hello"));
    fn();  // Output: 1 3.14 hello
    return 0;
}

```

### Q2: Compare the C++17 tuple workaround with C++20 pack init-captures

```cpp

#include <iostream>
#include <tuple>
#include <utility>

// C++17: wrap in tuple, unwrap with std::apply
template<typename... Args>
auto old_way(Args&&... args) {
    return [tup = std::make_tuple(std::forward<Args>(args)...)]() {
        std::apply([](const auto&... a) {
            ((std::cout << a << " "), ...);
            std::cout << "\n";
        }, tup);
    };
}

// C++20: direct pack capture
template<typename... Args>
auto new_way(Args&&... args) {
    return [...vals = std::forward<Args>(args)]() {
        ((std::cout << vals << " "), ...);
        std::cout << "\n";
    };
}

int main() {
    old_way(1, 2, 3)();  // 1 2 3
    new_way(1, 2, 3)();  // 1 2 3 — cleaner, no tuple overhead
    return 0;
}

```

### Q3: Implement a simple `bind_front` using pack init-captures

```cpp

#include <functional>
#include <iostream>
#include <utility>

template<typename F, typename... BoundArgs>
auto my_bind_front(F&& f, BoundArgs&&... bound) {
    return [f = std::forward<F>(f),
            ...front = std::forward<BoundArgs>(bound)]
           (auto&&... back) mutable
           -> decltype(auto) {
        return std::invoke(f, front...,
                           std::forward<decltype(back)>(back)...);
    };
}

void report(int id, const std::string& name, double score) {
    std::cout << "ID=" << id << " Name=" << name
              << " Score=" << score << "\n";
}

int main() {
    auto report_42 = my_bind_front(report, 42, std::string("Alice"));
    report_42(98.5);  // ID=42 Name=Alice Score=98.5
    return 0;
}

```

---

## Notes

- The syntax is `[...name = init-expr]` — the `...` precedes the capture name.
- Pack init-captures work with `mutable` lambdas and `constexpr` lambdas.
- This replaces the `std::tuple` + `std::apply` workaround from C++17.
- Be careful with capture-by-reference packs — lifetime issues apply just like single-variable captures.
