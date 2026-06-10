# Use algebraic data types with std::variant as sum type and std::tuple as product type

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

Algebraic data types (ADTs) form the foundation of typed functional programming. The name sounds academic, but the idea is concrete: a type is either "one of several things" (a sum type) or "all of several things at once" (a product type). C++ has both via `std::variant` for sum types and `std::tuple` for product types.

Understanding the distinction helps you model your problem domain precisely. Instead of using a base class with virtual functions when you have a fixed, known set of alternatives, you reach for `std::variant`. Instead of returning multiple values via out-parameters, you reach for `std::tuple` or a plain struct.

### Sum Types (variant = "one of")

A `std::variant<A, B, C>` holds exactly one of A, B, or C at any given time - never two, never none. The term "sum type" comes from the fact that the total number of possible values is A's values + B's values + C's values. Here's a practical example showing how to model an AST node:

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

The `unique_ptr<BinOp>` indirection is necessary because `Expr` is recursive - `BinOp` contains `Expr` values, so the type would be infinitely large without the pointer. This is a common pattern when modeling recursive structures with `std::variant`.

### Product Types (tuple = "all of")

A `std::tuple<A, B, C>` holds all of A, B, and C simultaneously. The term "product type" comes from the Cartesian product - the total number of possible values is A's values times B's values times C's values. C++17 structured bindings make tuples ergonomic to work with:

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

In practice, a plain struct is almost always clearer than a tuple because the fields have names. Tuples shine in generic code where you're working with types programmatically, or when returning multiple values from a function that you'll immediately destructure.

### Pattern Matching with std::visit

The real power of sum types appears when you combine them with `std::visit`. Every alternative must be handled - the compiler enforces exhaustiveness:

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

If you add a fourth shape type to the variant and forget to update `area`, the code will not compile. That is a very different guarantee from virtual dispatch, where a missing override compiles fine and silently produces wrong behavior at runtime.

---

## Self-Assessment

### Q1: How do sum types improve type safety over inheritance

With `std::variant`, all alternatives are known at compile time and `std::visit` forces you to handle every case - forgetting one is a compile error. With inheritance, adding a new derived class doesn't cause compile errors at call sites that use `dynamic_cast` or type switches, leading to silent runtime failures. The variant approach is "closed" (the set of alternatives is fixed) but provides strong compile-time guarantees; inheritance is "open" (new classes can always be added) but provides weaker guarantees.

### Q2: Show the relationship between product types and struct

A struct with N fields is a product type - its set of possible values is the Cartesian product of all field types. `std::tuple<int, bool>` has `INT_MAX * 2` possible values, the same as `struct { int x; bool y; }`. Structs are named product types with readable field access; tuples are anonymous product types with index-based or structured-binding access. The semantics are identical - the difference is ergonomics and documentation.

### Q3: Implement a recursive expression evaluator using variant

This builds directly on the `Expr` type defined earlier. The evaluator uses `std::visit` to dispatch on the node kind, and recursively evaluates sub-expressions for the binary operation case:

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

The recursive calls to `evaluate` work because `Expr` is defined before `evaluate`, even though `BinOp` contains `Expr` values. The `unique_ptr` indirection that makes the type compile also enables this recursion without any additional ceremony.

---

## Notes

- `std::variant` is a closed (fixed) sum type - all alternatives must be known at compile time. This is a deliberate restriction that buys you exhaustiveness checking.
- `std::visit` with the `overloaded` pattern is the C++ equivalent of pattern matching in functional languages.
- `std::variant` uses an integer index internally (not RTTI) making it zero-overhead compared to virtual dispatch.
- C++26 may add real pattern matching via the `inspect` statement, which will simplify variant usage considerably and remove the need for the `overloaded` helper.
