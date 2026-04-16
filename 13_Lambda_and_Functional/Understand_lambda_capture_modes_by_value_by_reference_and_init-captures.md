# Understand lambda capture modes: by value, by reference, and init-captures

**Category:** Lambda & Functional  
**Item:** #108  
**Standard:** C++11 (basic captures), C++14 (init-captures)  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

Lambda capture determines which outer variables a lambda can access and how. Three modes exist:

| Capture | Syntax | Effect |
| --- | --- | --- |
| By value | `[x]` or `[=]` | Copies `x` into closure at capture time |
| By reference | `[&x]` or `[&]` | Stores reference to `x`; sees mutations |
| Init-capture (C++14) | `[y = expr]` | Creates a new member initialized from `expr` |

```cpp

┌─────────────────────────────────────────────┐
│  [x, &y, z = std::move(p)]                 │
│   │   │    │                                │
│   │   │    └─ init-capture: moves p into z  │
│   │   └────── by reference: y is a ref      │
│   └────────── by value: x is a copy         │
└─────────────────────────────────────────────┘

```

### Capture Defaults

| Syntax | Meaning |
| --- | --- |
| `[=]` | Capture all used variables by value |
| `[&]` | Capture all used variables by reference |
| `[=, &x]` | All by value except `x` by reference |
| `[&, x]` | All by reference except `x` by value |

**Danger:** `[&]` can silently capture locals that outlive the lambda → dangling references.

---

## Self-Assessment

### Q1: Show a dangling reference bug in a lambda that captures a local variable by reference and is stored

**Solution:**

```cpp

#include <iostream>
#include <functional>

std::function<int()> make_counter_buggy() {
    int count = 0;
    // BUG: captures 'count' by reference, but 'count' is destroyed
    // when make_counter_buggy() returns!
    return [&count]() {
        return ++count;  // DANGLING REFERENCE — UB!
    };
}

std::function<int()> make_counter_fixed() {
    int count = 0;
    // FIX: capture by VALUE (with mutable to allow modification)
    return [count]() mutable {
        return ++count;  // OK — count is a member of the closure
    };
}

int main() {
    // BUGGY version — undefined behavior
    auto bad = make_counter_buggy();
    // std::cout << bad() << "\n";  // UB! 'count' no longer exists
    std::cout << "bad counter: UB (don't call!)\n";

    // FIXED version — safe
    auto good = make_counter_fixed();
    std::cout << "good counter: " << good() << "\n";  // 1
    std::cout << "good counter: " << good() << "\n";  // 2
    std::cout << "good counter: " << good() << "\n";  // 3

    // Another common bug: capturing loop variable by reference
    std::vector<std::function<int()>> funcs;
    for (int i = 0; i < 3; ++i) {
        funcs.push_back([&i]() { return i; });  // BUG: all share same 'i'
    }
    // After loop: i == 3 (or destroyed)
    // funcs[0]() would return 3, not 0!

    // FIX: capture by value
    funcs.clear();
    for (int i = 0; i < 3; ++i) {
        funcs.push_back([i]() { return i; });  // each gets its own copy
    }
    for (const auto& f : funcs)
        std::cout << f() << " ";
    std::cout << "\n";
}
// Expected output:
//   bad counter: UB (don't call!)
//   good counter: 1
//   good counter: 2
//   good counter: 3
//   0 1 2

```

---

### Q2: Use init-capture `[x = std::move(y)]` to move a `unique_ptr` into a lambda

**Solution:**

```cpp

#include <iostream>
#include <memory>
#include <functional>
#include <vector>

struct Resource {
    std::string name;
    Resource(std::string n) : name(std::move(n)) {
        std::cout << "Resource(" << name << ") created\n";
    }
    ~Resource() {
        std::cout << "Resource(" << name << ") destroyed\n";
    }
    void use() const { std::cout << "Using " << name << "\n"; }
};

int main() {
    auto res = std::make_unique<Resource>("GPU_Buffer");

    // Can't capture unique_ptr by value (no copy)!
    // auto bad = [res]() { res->use(); };  // ERROR: deleted copy ctor

    // Init-capture MOVES the unique_ptr into the lambda
    auto task = [r = std::move(res)]() {
        r->use();
    };

    // res is now nullptr — ownership transferred to lambda
    std::cout << "res is null: " << (res == nullptr) << "\n";

    task();  // lambda owns the resource

    // Practical use: store move-only lambda in a task queue
    // (requires std::function alternative or move-only wrapper)
    std::cout << "\n--- Init-capture with transform ---\n";
    std::vector<int> data = {1, 2, 3, 4, 5};
    int multiplier = 10;

    // Init-capture can rename and compute
    auto transform = [m = multiplier, base = data.size()](int x) {
        return x * m + static_cast<int>(base);
    };

    for (int x : data)
        std::cout << x << " → " << transform(x) << "\n";
}
// Expected output:
//   Resource(GPU_Buffer) created
//   res is null: 1
//   Using GPU_Buffer
//
//   --- Init-capture with transform ---
//   1 → 15
//   2 → 25
//   3 → 35
//   4 → 45
//   5 → 55
//   Resource(GPU_Buffer) destroyed

```

---

### Q3: Explain the difference between `[=]` and `[this]` for capturing in member functions

**Solution:**

```cpp

#include <iostream>
#include <functional>
#include <string>

struct Widget {
    std::string name = "Widget";
    int counter = 0;

    std::function<void()> get_lambda_this() {
        // [this] captures the 'this' POINTER by value
        // The lambda holds a raw pointer to 'this'
        // If Widget is destroyed → dangling pointer!
        return [this]() {
            std::cout << "[this] name=" << name
                      << " counter=" << ++counter << "\n";
        };
    }

    std::function<void()> get_lambda_eq() {
        // [=] in a member function captures 'this' by value (pointer copy!)
        // It does NOT copy the Widget members!
        // Same dangling risk as [this]!
        return [=]() {
            std::cout << "[=] name=" << name
                      << " counter=" << counter << "\n";
            // 'name' and 'counter' accessed through captured 'this' pointer
        };
    }

    std::function<void()> get_lambda_star_this() {
        // [*this] (C++17) captures the ENTIRE Widget BY VALUE
        // Safe even if original Widget is destroyed!
        return [*this]() mutable {
            std::cout << "[*this] name=" << name
                      << " counter=" << ++counter << "\n";
        };
    }
};

int main() {
    std::function<void()> f1, f2, f3;

    {
        Widget w;
        w.name = "Original";
        w.counter = 10;

        f1 = w.get_lambda_this();
        f2 = w.get_lambda_eq();
        f3 = w.get_lambda_star_this();

        f1();  // OK — w still alive
        f2();  // OK — w still alive
        f3();  // OK — has its own copy of w
    }
    // w is destroyed here!

    // f1();  // UB! 'this' pointer dangles!
    // f2();  // UB! also uses 'this' pointer!
    std::cout << "After Widget destroyed:\n";
    f3();     // Safe! Has its own copy
    f3();     // counter increments independently
}
// Expected output:
//   [this] name=Original counter=11
//   [=] name=Original counter=11
//   [*this] name=Original counter=11
//   After Widget destroyed:
//   [*this] name=Original counter=12
//   [*this] name=Original counter=13

```

**Summary table:**

| Capture | What's captured | Copies Widget data? | Dangling risk? |
| --- | --- | --- | --- |
| `[this]` | `this` pointer | No | **Yes** — if Widget dies |
| `[=]` | `this` pointer (implicitly) | No | **Yes** — same as `[this]`! |
| `[*this]` (C++17) | Entire Widget object | **Yes** | No — independent copy |

> **C++20 deprecation warning:** `[=]` implicitly capturing `this` is deprecated. Use `[=, this]` or `[=, *this]` explicitly.

---

## Notes

- **`mutable` keyword:** By-value captures are `const` by default. Use `mutable` to modify them.
- **Init-capture names:** `[x = expr]` creates a new name `x` in the closure — it doesn't need to match any outer variable.
- **`std::move` in init-capture:** `[v = std::move(vec)]` transfers ownership. The original is left in a moved-from state.
- **Pack capture (C++20):** `[...args = std::move(args)]` captures a parameter pack by move.
- **Reference lifetime:** A lambda captured by `[&]` never extends the lifetime of the referenced objects. Always ensure references outlive the lambda.
