# Implement pattern matching manually with std::visit + overload sets

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant/visit>  

---

## Topic Overview

C++ lacks native pattern matching (coming in C++26 with `inspect`), but `std::visit` with the overloaded pattern provides exhaustive, type-safe matching on variants. The technique requires a small helper, but once you have it, visiting a variant feels remarkably close to the pattern matching you'd write in Rust or Haskell - and it's just as safe.

### The Overloaded Pattern

The trick is a small helper struct that inherits from multiple lambdas simultaneously, bringing all of their `operator()` overloads into a single type. This lets `std::visit` do overload resolution across all of them for you:

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

That last comment is important: if `Value` had a fourth alternative and you didn't add a handler for it, the code would fail to compile. Compare that to a `switch` statement with a `default` case that silently ignores the new case at runtime.

### Multi-Variant Visit (Matching Pairs)

`std::visit` can accept multiple variants at once. The visitor then needs to handle all combinations of types across those variants. This is the C++ equivalent of matching on a tuple of values:

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

With two variants each having two alternatives, you need four handlers (two times two). For three variants you'd need eight, and so on. This can get unwieldy for large variant sets, which is part of what motivates the C++26 `inspect` proposal.

### AST Interpreter Example

A great real-world use case is AST evaluation, where each node type requires different handling. Here, the recursive structure of the expression tree maps naturally onto recursive `std::visit` calls:

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

The unique pointers are necessary here because `Expr` is a recursive type - `Add` contains `Expr` values, which are variants that can themselves contain `Add`. Without the pointer indirection, the type would be infinitely large. The rest is clean and readable: each handler pattern-matches on the node kind and does exactly the right thing.

---

## Self-Assessment

### Q1: Why is exhaustive matching important

If you add a new alternative to the variant, every `std::visit` call that handles it will produce a compile error until you add a handler. This prevents "forgotten case" bugs that plague if-else chains and switch statements, where `default` silently swallows new cases. With `std::visit`, the type system enforces that you've thought about every possibility.

### Q2: Show a generic catch-all visitor

Sometimes you genuinely want to handle "anything else" with a default action. You can do this with `auto&&`, but be aware of the trade-off - it turns exhaustive matching into non-exhaustive matching:

```cpp
std::visit(overloaded{
    [](int i)    { /* handle int */ },
    [](auto&&)   { /* catch-all for everything else */ },
}, value);
// auto&& matches any type not explicitly handled
// Use sparingly — it defeats exhaustiveness checking
```

### Q3: What will C++26 pattern matching look like

C++26's proposed `inspect` expression provides native syntax for the same thing, removing the need for the `overloaded` helper and making the intent much clearer:

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

The main advantages over `std::visit` are: no helper struct needed, structural patterns like `[x, y]`, guard conditions with `if`, and a wildcard `_` that's much more readable than `auto&&`.

---

## Notes

- The `overloaded` helper is so common it should be in a project utility header - you'll use it everywhere you have `std::variant`.
- `std::visit` has zero overhead - the compiler generates a jump table for dispatching to the right handler.
- C++20 CTAD makes the deduction guide unnecessary: `overloaded{...}` just works without the extra template declaration.
- For ad-hoc checks where you just need to test or extract one alternative, consider `std::holds_alternative` and `std::get_if` instead of setting up a full visitor.
