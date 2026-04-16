# Understand deducing this interaction with move semantics (C++23)

**Category:** Move Semantics and Value Categories  
**Standard:** C++23  
**Reference:** <https://wg21.link/P0847>  

---

## Topic Overview

C++23 "deducing this" (explicit object parameter) lets member functions deduce the value category of `*this`, eliminating the need for const/non-const/&&-qualified overload sets.

### Before C++23: Overload Explosion

```cpp

#include <string>
#include <optional>

class Widget {
    std::string name_;
public:
    // Need 3 overloads for different value categories:
    const std::string& name() const &  { return name_; }
    std::string&       name() &        { return name_; }
    std::string        name() &&       { return std::move(name_); }
};

```

### C++23: Single Function with Deducing This

```cpp

#include <string>
#include <iostream>

class Widget {
    std::string name_;
public:
    // ONE function handles all value categories:
    template<typename Self>
    auto&& name(this Self&& self) {
        return std::forward<Self>(self).name_;
    }
    // Deduces:
    // - Widget& → returns string&
    // - const Widget& → returns const string&
    // - Widget&& → returns string&&
};

int main() {
    Widget w;
    w.name() = "hello";                     // Returns string& (mutable lvalue)
    const Widget& cw = w;
    std::cout << cw.name() << "\n";         // Returns const string&
    auto s = Widget{}.name();               // Returns string&& (moved)
}

```

### CRTP Replacement

```cpp

// Before: CRTP for static polymorphism
template<typename Derived>
class Base {
public:
    void interface() {
        static_cast<Derived*>(this)->implementation();
    }
};

// C++23: deducing this replaces CRTP
class Base2 {
public:
    void interface(this auto&& self) {
        self.implementation();  // Deduces the actual derived type
    }
};

struct Derived2 : Base2 {
    void implementation() { std::cout << "Derived!\n"; }
};

```

---

## Self-Assessment

### Q1: How does `this auto&& self` deduce the value category

The explicit object parameter follows the same deduction rules as any forwarding reference. When called on an lvalue, `Self` deduces to `Widget&`; on a const lvalue, `const Widget&`; on an rvalue, `Widget`. `std::forward<Self>(self)` preserves the category.

### Q2: Can deducing this replace all ref-qualified overloads

For most patterns, yes. The single template replaces const/non-const/rvalue-ref overloads. Exception: when you need fundamentally different implementations per category (not just forwarding).

### Q3: Show how deducing this enables recursive lambdas

```cpp

auto factorial = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
std::cout << factorial(5) << "\n";  // 120
// The lambda can call itself via the explicit self parameter!

```

---

## Notes

- Deducing this eliminates the CRTP pattern for static polymorphism.
- Enables recursive lambdas without `std::function` or Y-combinator.
- Reduces boilerplate from 3-4 overloads to 1 function template.
- GCC 14+, Clang 18+, MSVC 19.36+ support deducing this.
