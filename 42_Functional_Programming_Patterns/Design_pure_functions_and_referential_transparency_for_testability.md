# Design pure functions and referential transparency for testability

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Referential_transparency>  

---

## Topic Overview

A **pure function** has no side effects and always returns the same output for the same input. Pure functions are trivially testable, parallelizable, and cacheable.

### Pure vs Impure

```cpp

#include <string>
#include <ctime>
#include <vector>
#include <numeric>

// IMPURE: depends on global state, has side effects
int counter = 0;
int impure_next() {
    return ++counter;  // Mutates global, different result each call
}

// IMPURE: depends on external state
std::string impure_greeting() {
    auto t = std::time(nullptr);
    auto* lt = std::localtime(&t);
    return lt->tm_hour < 12 ? "Good morning" : "Good afternoon";
    // Result depends on current time — not just inputs
}

// PURE: depends only on inputs, no side effects
double pure_average(const std::vector<double>& data) {
    if (data.empty()) return 0.0;
    return std::accumulate(data.begin(), data.end(), 0.0) / data.size();
    // Same input always gives same output
    // No global state modified
}

// PURE: string transformation
std::string to_slug(std::string_view title) {
    std::string result;
    for (char c : title) {
        if (std::isalnum(c)) result += std::tolower(c);
        else if (!result.empty() && result.back() != '-') result += '-';
    }
    if (!result.empty() && result.back() == '-') result.pop_back();
    return result;
}
// to_slug("Hello World!") always returns "hello-world"

```

### Making Impure Code Pure via Dependency Injection

```cpp

#include <functional>
#include <string>

// IMPURE: directly reads clock
std::string get_greeting_impure() {
    auto now = std::chrono::system_clock::now();
    // ... depends on real time
}

// PURE: clock is a parameter
std::string get_greeting_pure(std::chrono::system_clock::time_point now) {
    auto time = std::chrono::system_clock::to_time_t(now);
    auto* lt = std::localtime(&time);
    return lt->tm_hour < 12 ? "Good morning" : "Good afternoon";
}

// Testable:
void test_greeting() {
    auto morning = /* 2024-01-01 08:00 */;
    assert(get_greeting_pure(morning) == "Good morning");  // Deterministic!
}

```

---

## Self-Assessment

### Q1: Why are pure functions easier to test

No setup required (no mocking global state), no teardown needed (no side effects to clean up), deterministic (same input = same output), and parallelizable (tests can run simultaneously without interfering).

### Q2: Can constexpr functions be impure

No. `constexpr` functions must be evaluable at compile time, which prohibits I/O, global state mutation, and non-deterministic operations. `constexpr` is the strongest purity guarantee in C++.

### Q3: Show the "functional core, imperative shell" pattern

```cpp

// PURE CORE: all business logic is pure
struct Order { double total; std::string status; };
Order apply_discount(Order order, double pct) {
    order.total *= (1.0 - pct);
    return order;
}
Order mark_shipped(Order order) {
    order.status = "shipped";
    return order;
}

// IMPURE SHELL: IO at the boundaries
void process_order(Database& db, int order_id) {
    auto order = db.load_order(order_id);     // Impure: IO
    order = apply_discount(order, 0.1);        // Pure
    order = mark_shipped(order);               // Pure
    db.save_order(order);                       // Impure: IO
}
// Test the pure functions without any database!

```

---

## Notes

- C++ doesn't enforce purity (unlike Haskell), but you can follow the discipline.
- `constexpr` and `consteval` are the closest to enforced purity.
- The "functional core, imperative shell" pattern isolates side effects at the boundary.
- Pure functions enable memoization (cache results) and automatic parallelization.
