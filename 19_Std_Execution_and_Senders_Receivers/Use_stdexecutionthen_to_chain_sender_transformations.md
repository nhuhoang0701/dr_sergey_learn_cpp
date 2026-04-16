# Use std::execution::then to chain sender transformations

**Category:** std::execution & Senders/Receivers  
**Item:** #603  
**Standard:** C++20  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2300r7.html>  

---

## Topic Overview

`then()` transforms the **value** produced by a sender through a callable, producing a new sender with the transformed type. The pipeline is lazy — no work happens until connected and started.

```cpp

Type flow through then():

  just(42)        → sender<int>
  | then(to_str)  → sender<string>      (int → string)
  | then(length)  → sender<size_t>      (string → size_t)
  | then(double)  → sender<size_t>      (size_t → size_t)

```

---

## Self-Assessment

### Q1: Pipe through `then` and verify the value reaches the receiver

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    // Basic then:
    auto pipeline = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Transforming: " << x << " * 2\n";
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84

    // Multi-value then:
    auto multi = stdexec::just(10, 20)
        | stdexec::then([](int a, int b) {
            return a + b;  // receives ALL predecessor values
        });
    auto [sum] = stdexec::sync_wait(std::move(multi)).value();
    std::cout << "Sum: " << sum << '\n';  // 30

    // Manual connect + start (low-level):
    // auto op = stdexec::connect(sender, receiver);
    // stdexec::start(op);  // triggers the pipeline
    // sync_wait does this internally.
}

```

### Q2: `then()` is lazy — no work until connected and started

```cpp

#include <stdexec/execution.hpp>
#include <iostream>

int main() {
    int counter = 0;

    // Build the pipeline (no work yet!):
    auto pipeline = stdexec::just(42)
        | stdexec::then([&counter](int x) {
            ++counter;
            return x * 2;
        })
        | stdexec::then([&counter](int x) {
            ++counter;
            return x + 1;
        });

    std::cout << "After build:  counter = " << counter << '\n';  // 0

    // sync_wait connects and starts the pipeline:
    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();

    std::cout << "After run:    counter = " << counter << '\n';  // 2
    std::cout << "Result: " << result << '\n';  // 85
}
// Output:
// After build:  counter = 0
// After run:    counter = 2
// Result: 85

```

**Laziness enables:**

- Building pipelines without side effects
- Storing pipelines in data structures
- Choosing when/where to execute
- Inspecting completion signatures at compile time

### Q3: Chain three `then()` — type changes at each step

```cpp

#include <stdexec/execution.hpp>
#include <iostream>
#include <string>
#include <cmath>

int main() {
    auto pipeline = stdexec::just(42)        // sender<int>
        | stdexec::then([](int x) -> double {
            // Step 1: int -> double
            return std::sqrt(static_cast<double>(x));
        })                                    // sender<double>
        | stdexec::then([](double x) -> std::string {
            // Step 2: double -> string
            return "sqrt = " + std::to_string(x);
        })                                    // sender<string>
        | stdexec::then([](const std::string& s) -> std::size_t {
            // Step 3: string -> size_t
            std::cout << s << '\n';
            return s.size();
        });                                   // sender<size_t>

    auto [length] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "String length: " << length << '\n';
}
// Output:
// sqrt = 6.480741
// String length: 15

```

```cpp

Type transformation at each then():

  just(42)                  │ completion_signatures<set_value(int)>
     │                       │
  then([](int) -> double)  │ completion_signatures<set_value(double)>
     │                       │
  then([](double) -> str)  │ completion_signatures<set_value(string)>
     │                       │
  then([](str) -> size_t)  │ completion_signatures<set_value(size_t)>

All type deduction happens at COMPILE TIME.
The compiler knows the full type chain before any code runs.

```

---

## Notes

- `then` only operates on the value channel; errors skip `then` nodes.
- If the callable throws, the exception routes to `set_error`.
- `then` returns a move-only sender; use `split()` to share.
- For void predecessors: `then([]() { return 42; })`.
- `upon_error` and `upon_stopped` are the error/cancellation equivalents of `then`.
