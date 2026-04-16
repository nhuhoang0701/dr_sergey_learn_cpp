# Know When to Mark Functions `noexcept` and Its Performance Implications

**Category:** Move Semantics & Value Categories  
**Item:** #41  
**Reference:** <https://en.cppreference.com/w/cpp/language/noexcept_spec>  

---

## Topic Overview

### When to Use `noexcept`

| Mark `noexcept` | Don't mark `noexcept` |
| --- | --- |
| Move constructors/assignments | Functions that might throw |
| Swap functions | Functions with complex logic |
| Destructors (implicit already) | Virtual functions (may be overridden) |
| Leaf functions that can't throw | Functions calling unknown code |
| Comparison operators | Allocating functions (`new`, containers) |

### Why It Matters for Performance

1. **`std::vector` reallocation**: Uses `std::move_if_noexcept` — falls back to **copy** if move isn't `noexcept`
2. **Compiler optimizations**: `noexcept` allows removing exception-handling code paths
3. **Inlining**: No need to generate unwind tables for `noexcept` functions

### The Contract

`noexcept` is a hard contract: if a `noexcept` function throws, `std::terminate` is called immediately — no stack unwinding, no cleanup.

---

## Self-Assessment

### Q1: Add `noexcept` to a move constructor and verify `std::vector` uses move instead of copy during reallocation

```cpp

#include <iostream>
#include <vector>
#include <string>

// Widget WITH noexcept move → vector will MOVE during reallocation
class SafeWidget {
    std::string name_;
public:
    SafeWidget(std::string n) : name_(std::move(n)) {}

    SafeWidget(SafeWidget&& other) noexcept  // ← noexcept!
        : name_(std::move(other.name_)) {
        std::cout << "  MOVE: " << name_ << "\n";
    }

    SafeWidget(const SafeWidget& other) : name_(other.name_) {
        std::cout << "  COPY: " << name_ << "\n";
    }

    SafeWidget& operator=(SafeWidget&&) noexcept = default;
};

// Widget WITHOUT noexcept move → vector falls back to COPY
class UnsafeWidget {
    std::string name_;
public:
    UnsafeWidget(std::string n) : name_(std::move(n)) {}

    UnsafeWidget(UnsafeWidget&& other)  // ← NO noexcept!
        : name_(std::move(other.name_)) {
        std::cout << "  MOVE: " << name_ << "\n";
    }

    UnsafeWidget(const UnsafeWidget& other) : name_(other.name_) {
        std::cout << "  COPY: " << name_ << "\n";
    }
};

int main() {
    std::cout << "=== noexcept move → vector uses MOVE ===\n";
    {
        std::vector<SafeWidget> v;
        v.reserve(2);
        v.emplace_back("A");
        v.emplace_back("B");
        std::cout << "  Reallocation triggered:\n";
        v.emplace_back("C");  // Capacity exceeded → reallocation
        // A and B are MOVED (noexcept move available)
    }

    std::cout << "\n=== Non-noexcept move → vector uses COPY ===\n";
    {
        std::vector<UnsafeWidget> v;
        v.reserve(2);
        v.emplace_back("A");
        v.emplace_back("B");
        std::cout << "  Reallocation triggered:\n";
        v.emplace_back("C");
        // A and B are COPIED (move might throw → unsafe for strong guarantee)
    }

    // How vector decides internally:
    std::cout << "\n=== How std::vector decides ===\n";
    std::cout << "  std::move_if_noexcept(x) returns:\n";
    std::cout << "    T&& if T's move ctor is noexcept → MOVE\n";
    std::cout << "    const T& if T's move ctor may throw → COPY\n";
    std::cout << "  Reason: strong exception guarantee requires rollback\n";
    std::cout << "          if move throws mid-reallocation, elements are lost!\n";

    // Compile-time check
    std::cout << "\n  is_nothrow_move_constructible:\n";
    std::cout << "    SafeWidget:   " << std::is_nothrow_move_constructible_v<SafeWidget> << "\n";
    std::cout << "    UnsafeWidget: " << std::is_nothrow_move_constructible_v<UnsafeWidget> << "\n";

    return 0;
}

```

### Q2: Explain how `noexcept` affects exception specification inheritance and virtual dispatch

```cpp

#include <iostream>

// Rule: A derived class override can be MORE restrictive (add noexcept)
//       but CANNOT be LESS restrictive (remove noexcept from base).

class Base {
public:
    virtual ~Base() = default;

    // noexcept virtual function
    virtual void safe_op() noexcept {
        std::cout << "  Base::safe_op (noexcept)\n";
    }

    // Potentially throwing virtual function
    virtual void risky_op() {
        std::cout << "  Base::risky_op (may throw)\n";
    }
};

class Derived : public Base {
public:
    // ✅ OK: Base is noexcept, Derived is also noexcept (matching)
    void safe_op() noexcept override {
        std::cout << "  Derived::safe_op (noexcept)\n";
    }

    // ❌ COMPILE ERROR if uncommented:
    // void safe_op() override {  // ERROR: looser exception spec
    //     throw std::runtime_error("oops");
    // }

    // ✅ OK: Base may throw, Derived promises noexcept (more restrictive)
    void risky_op() noexcept override {
        std::cout << "  Derived::risky_op (now noexcept)\n";
    }
};

// Exception specification inheritance:
// - Implicitly declared special members inherit noexcept from base/members
// - If all bases/members have noexcept move → derived gets noexcept move

struct A { A(A&&) noexcept {} };
struct B { B(B&&) noexcept {} };
struct C : A {
    B member;
    // C(C&&) is implicitly noexcept (A move + B move are both noexcept)
};

struct D { D(D&&) {} };  // NOT noexcept
struct E : A {
    D member;
    // E(E&&) is implicitly NOT noexcept (D move may throw)
};

int main() {
    std::cout << "=== noexcept and Virtual Dispatch ===\n\n";

    Derived d;
    Base& b = d;

    b.safe_op();   // Calls Derived::safe_op (noexcept guarantee preserved)
    b.risky_op();  // Calls Derived::risky_op (now noexcept, but caller doesn't know)

    std::cout << "\n=== Exception Spec Inheritance ===\n";
    std::cout << "  C move noexcept: " << std::is_nothrow_move_constructible_v<C> << "\n";  // 1
    std::cout << "  E move noexcept: " << std::is_nothrow_move_constructible_v<E> << "\n";  // 0

    std::cout << "\n=== Rules ===\n";
    std::cout << "  1. Override can add noexcept (more restrictive) ✅\n";
    std::cout << "  2. Override cannot remove noexcept (less restrictive) ❌\n";
    std::cout << "  3. Implicit noexcept = AND of all sub-objects' noexcept\n";

    return 0;
}

```

### Q3: Show that calling a `noexcept` function that throws invokes `std::terminate`

```cpp

#include <iostream>
#include <exception>
#include <cstdlib>

// Custom terminate handler to prove it's called
void my_terminate() {
    std::cout << "  *** std::terminate called! ***\n";
    std::cout << "  No stack unwinding occurred.\n";
    std::cout << "  No catch blocks were checked.\n";
    std::abort();  // Must terminate the program
}

// A noexcept function that THROWS — this is undefined-ish behavior
// The standard says: std::terminate is called
void dangerous() noexcept {
    std::cout << "  In dangerous() — about to throw...\n";
    throw std::runtime_error("oops");
    // This throw WILL NOT propagate!
    // std::terminate is called IMMEDIATELY
}

int main() {
    // Install custom terminate handler
    std::set_terminate(my_terminate);

    std::cout << "=== noexcept + throw = std::terminate ===\n\n";

    try {
        std::cout << "Inside try block, calling dangerous()...\n";
        dangerous();
        std::cout << "After dangerous() — NEVER REACHED\n";
    }
    catch (const std::exception& e) {
        // This catch block is NEVER entered!
        std::cout << "Caught: " << e.what() << " — NEVER REACHED\n";
    }
    catch (...) {
        // This catch block is NEVER entered either!
        std::cout << "Caught unknown — NEVER REACHED\n";
    }

    std::cout << "After try/catch — NEVER REACHED\n";
    return 0;
}
// Expected output:
//   === noexcept + throw = std::terminate ===
//   Inside try block, calling dangerous()...
//   In dangerous() — about to throw...
//   *** std::terminate called! ***
//   No stack unwinding occurred.
//   No catch blocks were checked.
//   (program aborted)
//
// KEY: The exception does NOT propagate to any catch block.
// std::terminate is called before unwinding begins.

```

---

## Notes

- **Always** mark move constructors, move assignment, swap, and destructors `noexcept`.
- `noexcept` is part of the type system since C++17 — `void(*)() noexcept` ≠ `void(*)()`.
- Use `noexcept(expr)` for conditional noexcept: `noexcept(noexcept(other_func()))`.
- Don't add `noexcept` to functions that might allocate memory — `new` can throw `bad_alloc`.
- Compiler flag `-Wnoexcept` (GCC) warns when `noexcept` would improve performance.
