# Master Perfect Forwarding with std::forward and Universal References

**Category:** Move Semantics & Value Categories  
**Item:** #39  
**Reference:** <https://en.cppreference.com/w/cpp/utility/forward>  

---

## Topic Overview

### What Is Perfect Forwarding

Perfect forwarding passes arguments to another function **preserving their value category** (lvalue/rvalue) and **cv-qualification** (const/non-const). Without it, wrapper functions degrade rvalues to lvalues, causing unnecessary copies.

### Reference Collapsing Rules

| `T` deduced as | `T&&` becomes | Result |
| --- | --- | --- |
| `Widget` | `Widget&&` | rvalue reference |
| `Widget&` | `Widget& &&` → `Widget&` | lvalue reference |
| `Widget&&` | `Widget&& &&` → `Widget&&` | rvalue reference |
| `const Widget&` | `const Widget& &&` → `const Widget&` | const lvalue reference |

**Rule:** `& + anything = &`. Only `&& + && = &&`.

### The Pattern

```cpp

template <typename T>
void wrapper(T&& arg) {                // Forwarding reference (T is deduced)
    target(std::forward<T>(arg));       // Preserves value category
}

// What std::forward does:
// If T = Widget&  → forward returns Widget&  (lvalue)
// If T = Widget   → forward returns Widget&& (rvalue)

```

### Forwarding Reference vs Rvalue Reference

```cpp

template <typename T>
void f(T&& x);          // FORWARDING reference (T is deduced from template)

void g(Widget&& x);     // RVALUE reference (Widget is a concrete type)

```

---

## Self-Assessment

### Q1: Write a `make` function that perfectly forwards all arguments to a constructor

```cpp

#include <iostream>
#include <string>
#include <utility>
#include <memory>

struct Widget {
    std::string name;
    int value;
    double weight;

    Widget(std::string n, int v, double w)
        : name(std::move(n)), value(v), weight(w) {
        std::cout << "  Constructed Widget(\"" << name << "\", "
                  << value << ", " << weight << ")\n";
    }

    Widget(const Widget& other)
        : name(other.name), value(other.value), weight(other.weight) {
        std::cout << "  COPY Widget\n";
    }

    Widget(Widget&& other) noexcept
        : name(std::move(other.name)), value(other.value), weight(other.weight) {
        std::cout << "  MOVE Widget\n";
    }
};

// Perfect forwarding factory — works with any argument types
template <typename T, typename... Args>
T make(Args&&... args) {
    return T(std::forward<Args>(args)...);  // Forward all args with their categories
}

// Also works for unique_ptr-style wrappers
template <typename T, typename... Args>
std::unique_ptr<T> make_unique_custom(Args&&... args) {
    return std::unique_ptr<T>(new T(std::forward<Args>(args)...));
}

int main() {
    std::cout << "=== Perfect Forwarding Factory ===\n\n";

    // All arguments forwarded perfectly
    std::cout << "1. All rvalues:\n";
    auto w1 = make<Widget>(std::string("Alpha"), 42, 3.14);

    std::cout << "\n2. Mixed lvalue/rvalue:\n";
    std::string name = "Beta";
    auto w2 = make<Widget>(name, 100, 2.71);  // name forwarded as lvalue → copied
    std::cout << "  name after: \"" << name << "\" (unchanged — was lvalue)\n";

    std::cout << "\n3. Move an lvalue explicitly:\n";
    auto w3 = make<Widget>(std::move(name), 200, 1.41);
    std::cout << "  name after: \"" << name << "\" (moved from)\n";

    std::cout << "\n4. unique_ptr factory with forwarding:\n";
    auto p = make_unique_custom<Widget>(std::string("Gamma"), 300, 0.5);

    return 0;
}
// Expected output:
//   1. All rvalues:
//     Constructed Widget("Alpha", 42, 3.14)
//   2. Mixed lvalue/rvalue:
//     Constructed Widget("Beta", 100, 2.71)
//     name after: "Beta" (unchanged — was lvalue)
//   3. Move an lvalue explicitly:
//     Constructed Widget("Beta", 200, 1.41)
//     name after: "" (moved from)
//   4. unique_ptr factory with forwarding:
//     Constructed Widget("Gamma", 300, 0.5)

```

### Q2: Explain the reference collapsing rules that make `T&&` a forwarding reference when `T` is deduced

When `T` is a **deduced template parameter**, `T&&` is a *forwarding reference* (also called *universal reference*). The magic comes from **reference collapsing**:

```cpp

#include <iostream>
#include <type_traits>

// When T is deduced from the argument:
// - If argument is lvalue of type X, T = X& → T&& = X& && → collapses to X&
// - If argument is rvalue of type X, T = X  → T&& = X&&   → stays X&&

template <typename T>
void show_type(T&& arg) {
    if constexpr (std::is_lvalue_reference_v<T>) {
        std::cout << "  T is lvalue ref → T&& collapsed to lvalue ref\n";
    } else {
        std::cout << "  T is non-ref → T&& is rvalue ref\n";
    }

    // std::forward uses T to decide:
    // If T = X&  → forward returns X& (lvalue)
    // If T = X   → forward returns X&& (rvalue)
}

// Reference collapsing table:
//   T = int&    → T&& = int& && = int&     (& wins)
//   T = int&&   → T&& = int&& && = int&&   (&& wins)
//   T = int     → T&& = int&&              (simple)
//   T = const int& → T&& = const int&      (& wins, const preserved)

void takes_lref(int&)  { std::cout << "  → lvalue ref overload\n"; }
void takes_rref(int&&) { std::cout << "  → rvalue ref overload\n"; }

template <typename T>
void perfect_forward(T&& arg) {
    show_type(std::forward<T>(arg));
}

int main() {
    std::cout << "=== Reference Collapsing Demo ===\n\n";

    int x = 42;

    std::cout << "Passing lvalue (x):\n";
    perfect_forward(x);       // T = int& → T&& = int&

    std::cout << "Passing rvalue (42):\n";
    perfect_forward(42);      // T = int → T&& = int&&

    std::cout << "Passing std::move(x):\n";
    perfect_forward(std::move(x));  // T = int → T&& = int&&

    const int cx = 10;
    std::cout << "Passing const lvalue:\n";
    perfect_forward(cx);      // T = const int& → T&& = const int&

    return 0;
}

```

### Q3: Show a case where forgetting `std::forward` causes an extra copy in an emplace-style wrapper

```cpp

#include <iostream>
#include <string>
#include <vector>
#include <utility>

struct Entry {
    std::string key;
    std::string value;

    Entry(std::string k, std::string v)
        : key(std::move(k)), value(std::move(v)) {
        std::cout << "  Entry(\"" << key << "\", \"" << value << "\")\n";
    }
    Entry(const Entry& other) : key(other.key), value(other.value) {
        std::cout << "  COPY Entry\n";
    }
    Entry(Entry&& other) noexcept
        : key(std::move(other.key)), value(std::move(other.value)) {
        std::cout << "  MOVE Entry\n";
    }
};

class Registry {
    std::vector<Entry> entries_;
public:
    // BUGGY: Forgets std::forward → args are always lvalues → extra copies!
    template <typename... Args>
    void emplace_buggy(Args&&... args) {
        entries_.emplace_back(args...);  // BUG: args are lvalues (named params)
        // Even though caller passed rvalues, they arrive as lvalues here
    }

    // CORRECT: Uses std::forward to preserve value categories
    template <typename... Args>
    void emplace_correct(Args&&... args) {
        entries_.emplace_back(std::forward<Args>(args)...);  // Perfect forwarding
    }
};

int main() {
    std::cout << "=== Forgetting std::forward ===\n\n";

    Registry r;

    std::cout << "BUGGY (no forward) — rvalue strings get COPIED:\n";
    r.emplace_buggy(std::string("key1"), std::string("val1"));
    // The string temporaries bind to Args&&... as rvalues,
    // but inside emplace_buggy, 'args' are NAMED → lvalues → string copies!

    std::cout << "\nCORRECT (with forward) — rvalue strings get MOVED:\n";
    r.emplace_correct(std::string("key2"), std::string("val2"));
    // std::forward<Args>(args)... restores the rvalue-ness → string moves!

    std::cout << "\nWith lvalues (both behave the same — lvalue → copy):\n";
    std::string k = "key3", v = "val3";
    r.emplace_correct(k, v);  // lvalues → correctly forwarded as lvalues → copies

    return 0;
}
// Expected output:
//   BUGGY (no forward) — rvalue strings get COPIED:
//     Entry("key1", "val1")              ← strings were copied into Entry
//   CORRECT (with forward) — rvalue strings get MOVED:
//     Entry("key2", "val2")              ← strings were moved into Entry
//   With lvalues (both behave the same):
//     Entry("key3", "val3")              ← strings were copied (correct for lvalues)

```

---

## Notes

- `T&&` is a **forwarding reference** only when `T` is deduced — `Widget&&` is always an rvalue reference.
- Always use `std::forward<T>(arg)`, never `std::move` inside a forwarding function (unless you want to force a move).
- `std::forward` is a **conditional cast**: it casts to rvalue only if the original argument was an rvalue.
- Named parameters are always lvalues — that's why `std::forward` is needed to restore the original category.
- In C++20, use `auto&&` parameters in lambdas as forwarding references with `std::forward<decltype(arg)>(arg)`.
