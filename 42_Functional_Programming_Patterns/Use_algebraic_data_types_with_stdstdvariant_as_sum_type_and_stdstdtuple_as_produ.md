# Use algebraic data types with std::variant as sum type and std::tuple as product type

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

Algebraic data types (ADTs) form the foundation of typed functional programming. C++ has them via `std::variant` (sum types / tagged unions) and `std::tuple` (product types).

### Sum Types (variant = "one of")

```cpp

#include <variant>
#include <string>
#include <iostream>
#include <vector>

// A JSON value is ONE of these types:
using JsonValue = std::variant<
    std::nullptr_t,           // null
    bool,                     // boolean
    double,                   // number
    std::string,              // string
    std::vector<JsonValue>,   // array  (forward-declared via recursive_wrapper)
    std::map<std::string, JsonValue>  // object
>;
// This is not directly possible due to incomplete type — use a wrapper or box

// Simpler example: a compiler AST node
struct Literal { double value; };
struct Variable { std::string name; };
struct BinOp;

using Expr = std::variant<Literal, Variable, std::unique_ptr<BinOp>>;

struct BinOp {
    char op;
    Expr left, right;
};

```

### Product Types (tuple = "all of")

```cpp

#include <tuple>
#include <string>

// A product type holds ALL of these simultaneously:
using Person = std::tuple<std::string, int, double>;  // name, age, height

// Structured bindings make product types ergonomic:
auto [name, age, height] = Person{"Alice", 30, 5.7};

// Named alternative (usually preferred):
struct PersonNamed {
    std::string name;
    int age;
    double height;
};

```

### Pattern Matching with std::visit

```cpp

#include <variant>
#include <string>
#include <iostream>

// Overload pattern for visitor
template<class... Ts> struct overloaded : Ts... { using Ts::operator()...; };

using Shape = std::variant<Circle, Rectangle, Triangle>;

double area(const Shape& shape) {
    return std::visit(overloaded{
        [](const Circle& c)    { return 3.14159 * c.radius * c.radius; },
        [](const Rectangle& r) { return r.width * r.height; },
        [](const Triangle& t)  { return 0.5 * t.base * t.height; },
    }, shape);
}

```

---

## Self-Assessment

### Q1: How do sum types improve type safety over inheritance

With `std::variant`, all alternatives are known at compile time. `std::visit` forces you to handle every case — forgetting one is a compile error. With inheritance, adding a new derived class doesn't cause compile errors at call sites that use `dynamic_cast` or type switches, leading to runtime failures.

### Q2: Show the relationship between product types and struct

A struct with N fields IS a product type — its set of possible values is the Cartesian product of all field types. `std::tuple<int, bool>` has `INT_MAX * 2` possible values, same as `struct { int x; bool y; }`. Structs are named product types; tuples are anonymous.

### Q3: Implement a recursive expression evaluator using variant

```cpp

double evaluate(const Expr& expr) {
    return std::visit(overloaded{
        [](const Literal& lit) { return lit.value; },
        [](const Variable& var) -> double {
            throw std::runtime_error("unbound variable: " + var.name);
        },
        [](const std::unique_ptr<BinOp>& op) -> double {
            double l = evaluate(op->left);
            double r = evaluate(op->right);
            switch (op->op) {
                case '+': return l + r;
                case '-': return l - r;
                case '*': return l * r;
                case '/': return l / r;
                default: throw std::runtime_error("unknown op");
            }
        },
    }, expr);
}

```

---

## Notes

- `std::variant` is a closed (fixed) sum type — all alternatives must be known at compile time.
- `std::visit` with the `overloaded` pattern is the C++ equivalent of pattern matching.
- `std::variant` uses an index (not RTTI) — it's zero-overhead compared to virtual dispatch.
- C++26 may add real pattern matching (`inspect` statement) which simplifies variant usage.
