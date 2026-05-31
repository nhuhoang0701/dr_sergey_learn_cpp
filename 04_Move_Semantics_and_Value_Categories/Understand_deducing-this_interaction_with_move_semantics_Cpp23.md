# Understand deducing this interaction with move semantics (C++23)

**Category:** Move Semantics and Value Categories  
**Standard:** C++23  
**Reference:** <https://wg21.link/P0847>  

---

## Topic Overview

C++23 "deducing this" (explicit object parameter) lets member functions deduce the value category of `*this`, eliminating the need for const/non-const/&&-qualified overload sets.

### Before C++23: Overload Explosion

Before C++23, if you wanted a member function to behave correctly for lvalue, const-lvalue, and rvalue objects, you had to write three separate overloads. Here is what that looks like:

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

Three functions doing essentially the same thing - just forwarding `name_` with the right value category. That is a lot of repetition for a simple accessor.

### C++23: Single Function with Deducing This

C++23 introduces the explicit object parameter (`this Self&& self`), which lets the compiler deduce `Self` the same way it deduces `T` in a forwarding reference. The result is one template that handles all three cases automatically. Notice how `std::forward<Self>(self).name_` propagates the correct value category:

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
    // - Widget& -> returns string&
    // - const Widget& -> returns const string&
    // - Widget&& -> returns string&&
};

int main() {
    Widget w;
    w.name() = "hello";                     // Returns string& (mutable lvalue)
    const Widget& cw = w;
    std::cout << cw.name() << "\n";         // Returns const string&
    auto s = Widget{}.name();               // Returns string&& (moved)
}
```

The deduction follows the exact same rules as a forwarding reference in a free function template. If `self` binds to a `Widget&`, `Self` deduces to `Widget&`; if it binds to a `Widget&&`, `Self` deduces to `Widget`. Then `std::forward<Self>(self)` does the right thing in each case.

### CRTP Replacement

The same mechanism that makes accessors simpler also replaces the CRTP (Curiously Recurring Template Pattern) for static polymorphism. The old CRTP approach required a `static_cast` to the derived type. With deducing this, `self` already carries the actual derived type - no cast needed:

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

The `this auto&& self` shorthand is equivalent to `template<typename Self> void interface(this Self&& self)` - it is just more concise.

---

## Self-Assessment

### Q1: How does `this auto&& self` deduce the value category

The explicit object parameter follows the same deduction rules as any forwarding reference. When called on an lvalue, `Self` deduces to `Widget&`; on a const lvalue, `const Widget&`; on an rvalue, `Widget`. `std::forward<Self>(self)` preserves the category.

### Q2: Can deducing this replace all ref-qualified overloads

For most patterns, yes. The single template replaces const/non-const/rvalue-ref overloads. Exception: when you need fundamentally different implementations per category (not just forwarding).

### Q3: Show how deducing this enables recursive lambdas

The reason recursive lambdas were awkward before C++23 is that a lambda has no name to call itself by. The explicit object parameter gives it one - `self` inside the lambda body refers to the lambda itself, so it can recurse naturally:

```cpp
auto factorial = [](this auto self, int n) -> int {
    return n <= 1 ? 1 : n * self(n - 1);
};
std::cout << factorial(5) << "\n";  // 120
// The lambda can call itself via the explicit self parameter!
```

This avoids the old workarounds (`std::function` with type erasure overhead, or the Y-combinator trick).

---

## Notes

- Deducing this eliminates the CRTP pattern for static polymorphism.
- Enables recursive lambdas without `std::function` or Y-combinator.
- Reduces boilerplate from 3-4 overloads to 1 function template.
- GCC 14+, Clang 18+, MSVC 19.36+ support deducing this.
