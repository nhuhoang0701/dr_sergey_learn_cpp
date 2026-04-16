# Implement pattern matching manually with std::visit + overload sets

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant/visit>  

---

## Topic Overview

C++ lacks native pattern matching (coming in C++26 with `inspect`), but `std::visit` with the overloaded pattern provides exhaustive, type-safe matching on variants.

### The Overloaded Pattern

```cpp

#include <variant>
#include <string>
#include <iostream>

// C++17 helper: combine multiple lambdas into one visitor
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

// C++17 deduction guide (not needed in C++20):
template<class... Ts> overloaded(Ts...) -> overloaded<Ts...>;

// Usage:
using Value = std::variant<int, double, std::string>;

void print_value(const Value& v) {
    std::visit(overloaded{
        [](int i)                { std::cout << "int: " << i << "\n"; },
        [](double d)             { std::cout << "double: " << d << "\n"; },
        [](const std::string& s) { std::cout << "string: " << s << "\n"; },
    }, v);
}
// Compile error if you forget a case! (exhaustive matching)

```

### Multi-Variant Visit (Matching Pairs)

```cpp

using Operand = std::variant<int, double>;

Operand add(const Operand& a, const Operand& b) {
    return std::visit(overloaded{
        [](int x, int y)       -> Operand { return x + y; },
        [](double x, double y) -> Operand { return x + y; },
        [](int x, double y)    -> Operand { return x + y; },
        [](double x, int y)    -> Operand { return x + y; },
    }, a, b);
}

```

### AST Interpreter Example

```cpp

#include <variant>
#include <memory>
#include <string>
#include <iostream>
#include <cmath>

struct Number { double value; };
struct Add;
struct Mul;

using Expr = std::variant<Number, std::unique_ptr<Add>, std::unique_ptr<Mul>>;

struct Add { Expr left, right; };
struct Mul { Expr left, right; };

double eval(const Expr& e) {
    return std::visit(overloaded{
        [](const Number& n) { return n.value; },
        [](const std::unique_ptr<Add>& a) {
            return eval(a->left) + eval(a->right);
        },
        [](const std::unique_ptr<Mul>& m) {
            return eval(m->left) * eval(m->right);
        },
    }, e);
}

```

---

## Self-Assessment

### Q1: Why is exhaustive matching important

If you add a new alternative to the variant, every `std::visit` call that handles it will produce a compile error until you add a handler. This prevents "forgotten case" bugs that plague if-else chains and switch statements (where `default` silently swallows new cases).

### Q2: Show a generic catch-all visitor

```cpp

std::visit(overloaded{
    [](int i)    { /* handle int */ },
    [](auto&&)   { /* catch-all for everything else */ },
}, value);
// auto&& matches any type not explicitly handled
// Use sparingly — it defeats exhaustiveness checking

```

### Q3: What will C++26 pattern matching look like

```cpp

// Proposed syntax (P2688):
inspect (value) {
    i as int           => std::cout << "int: " << i;
    d as double        => std::cout << "double: " << d;
    [x, y]            => std::cout << "pair: " << x << ", " << y;
    s if (s.empty())  => std::cout << "empty string";
    _                 => std::cout << "other";
};
// inspect is an expression — can return values

```

---

## Notes

- The `overloaded` helper is so common it should be in a project utility header.
- `std::visit` has zero overhead — the compiler generates a jump table.
- C++20 CTAD makes the deduction guide unnecessary: `overloaded{...}` just works.
- For complex matching, consider `std::holds_alternative` + `std::get_if` for ad-hoc checks.
