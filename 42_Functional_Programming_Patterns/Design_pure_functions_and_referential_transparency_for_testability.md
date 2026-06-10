# Design pure functions and referential transparency for testability

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Referential_transparency>  

---

## Topic Overview

A **pure function** has no side effects and always returns the same output for the same input. Pure functions are trivially testable, parallelizable, and cacheable. C++ doesn't enforce purity the way Haskell does, but you can absolutely apply the discipline - and when you do, your code becomes dramatically easier to reason about and test.

The term **referential transparency** means something closely related: an expression is referentially transparent if you can replace it with its value without changing the program's behavior. `pure_average({1.0, 2.0, 3.0})` is always `2.0`, so you could substitute the result anywhere it appears. `impure_next()` is not - calling it twice gives two different answers.

### Pure vs Impure

The distinction matters most when you sit down to write a test. Pure functions need no setup, no mocking, and no teardown. Impure functions require you to control or simulate the external state they depend on. Here are both in contrast:

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

Notice that `pure_average` and `to_slug` are completely self-contained. You can call them from any thread, at any time, in any order, with the same result. `impure_next` and `impure_greeting` cannot make that promise - their output depends on state that lives outside the function.

### Making Impure Code Pure via Dependency Injection

Here is the key technique: if a function needs external state to do its job, pass that state as a parameter instead of reaching out to grab it. The function stays pure; the caller decides what value to supply. This is sometimes called dependency injection, but the core idea is simple - make dependencies explicit arguments.

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

The impure version is untestable without controlling the system clock. The pure version lets any test supply any time point it wants, making the test deterministic and reliable.

---

## Self-Assessment

### Q1: Why are pure functions easier to test

No setup required (no mocking of global state), no teardown needed (no side effects to clean up), deterministic (same input always means same output), and parallelizable (tests can run simultaneously without interfering with each other). You can run pure function tests in any order, any number of times, with full confidence in the results.

### Q2: Can constexpr functions be impure

No. `constexpr` functions must be evaluable at compile time, which prohibits I/O, global state mutation, and non-deterministic operations. `constexpr` is the strongest purity guarantee C++ offers - if the compiler can evaluate it at compile time, it is pure by definition.

### Q3: Show the "functional core, imperative shell" pattern

The idea here is to push all the messy I/O and state mutation to the edges of your program, and keep the business logic in pure functions in the middle. The pure core is easy to test in isolation; the thin imperative shell wires it up to the real world:

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

You can write exhaustive unit tests for `apply_discount` and `mark_shipped` without touching a database at all. The impure shell is thin enough that you typically integration-test it rather than unit-test it.

---

## Notes

- C++ doesn't enforce purity (unlike Haskell), but you can follow the discipline and get the benefits.
- `constexpr` and `consteval` are the closest to enforced purity that C++ provides.
- The "functional core, imperative shell" pattern isolates side effects at the program boundary, making the majority of your code easy to test.
- Pure functions enable memoization (cache the result for a given input and skip recomputation) and automatic parallelization (no shared mutable state means no data races).
