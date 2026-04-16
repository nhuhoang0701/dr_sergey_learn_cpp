# Know when to use virtual functions vs std::variant vs CRTP for polymorphism

**Category:** OOP Design

---

## Topic Overview

C++ offers three distinct polymorphism mechanisms, each with radically different trade-offs:

| Aspect | Virtual Functions | std::variant | CRTP |
| --- | :---: | :---: | :---: |
| Dispatch time | Runtime | Runtime (visit) | **Compile-time** |
| Open/Closed set | **Open** (add types freely) | Closed (fixed type list) | Open (templates) |
| Heap required | Yes (polymorphic ptr) | **No** (inline storage) | **No** |
| Heterogeneous container | `vector<unique_ptr<Base>>` | `vector<variant>` | **No** (different types) |
| Adding new types | **Easy** (new derived class) | Recompile all visitors | **Easy** |
| Adding new operations | Hard (modify base) | **Easy** (new visitor) | Moderate |
| vtable overhead | ~8 bytes/object + indirection | Size = max alternative + tag | **Zero** |

### Decision Flowchart

```cpp

Is the set of types known at compile time?
├─ YES → Do you need a container of mixed types?
│        ├─ YES → std::variant + std::visit
│        └─ NO  → CRTP or simple templates
└─ NO  → Virtual functions (open set)

```

---

## Self-Assessment

### Q1: Implement the same problem with all three approaches

**Answer:**

```cpp

#include <variant>
#include <vector>
#include <memory>
#include <iostream>
#include <cmath>

// ──── APPROACH 1: Virtual functions ────
namespace virt {
    class Shape {
    public:
        virtual ~Shape() = default;
        virtual double area() const = 0;
    };
    class Circle : public Shape {
        double r_;
    public:
        explicit Circle(double r) : r_(r) {}
        double area() const override { return M_PI * r_ * r_; }
    };
    class Rect : public Shape {
        double w_, h_;
    public:
        Rect(double w, double h) : w_(w), h_(h) {}
        double area() const override { return w_ * h_; }
    };
    double total_area(const std::vector<std::unique_ptr<Shape>>& shapes) {
        double sum = 0;
        for (const auto& s : shapes) sum += s->area();
        return sum;
    }
}

// ──── APPROACH 2: std::variant ────
namespace var {
    struct Circle { double r; };
    struct Rect { double w, h; };
    using Shape = std::variant<Circle, Rect>;

    double area(const Shape& s) {
        return std::visit([](const auto& shape) -> double {
            using T = std::decay_t<decltype(shape)>;
            if constexpr (std::is_same_v<T, Circle>)
                return M_PI * shape.r * shape.r;
            else
                return shape.w * shape.h;
        }, s);
    }
    double total_area(const std::vector<Shape>& shapes) {
        double sum = 0;
        for (const auto& s : shapes) sum += area(s);
        return sum;
    }
}

// ──── APPROACH 3: CRTP ────
namespace crtp {
    template<typename Derived>
    class Shape {
    public:
        double area() const {
            return static_cast<const Derived*>(this)->area_impl();
        }
    };
    class Circle : public Shape<Circle> {
        double r_;
        friend class Shape<Circle>;
        double area_impl() const { return M_PI * r_ * r_; }
    public:
        explicit Circle(double r) : r_(r) {}
    };
    // Can't store Circle and Rect in same container!
    template<typename ShapeT>
    double process(const Shape<ShapeT>& s) {
        return s.area();  // Zero-overhead dispatch
    }
}

int main() {
    // Virtual: heap-allocated, open set
    std::vector<std::unique_ptr<virt::Shape>> v1;
    v1.push_back(std::make_unique<virt::Circle>(5));
    v1.push_back(std::make_unique<virt::Rect>(3, 4));
    std::cout << "Virtual: " << virt::total_area(v1) << "\n";

    // Variant: value semantics, cache friendly, closed set
    std::vector<var::Shape> v2 = { var::Circle{5}, var::Rect{3, 4} };
    std::cout << "Variant: " << var::total_area(v2) << "\n";

    // CRTP: zero overhead, no mixed containers
    crtp::Circle c(5);
    std::cout << "CRTP: " << crtp::process(c) << "\n";
    return 0;
}

```

### Q2: Show where each approach is the clear winner

**Answer:**

```cpp

// VIRTUAL wins: Plugin system (open set, loaded at runtime)
class AudioPlugin {
public:
    virtual ~AudioPlugin() = default;
    virtual void process(float* buf, size_t n) = 0;
};
// New plugins via dlopen() — set is OPEN

// VARIANT wins: Compiler AST (closed set, many operations)
using Expr = std::variant<IntLit, FloatLit, BinOp, UnaryOp, Ident>;
std::string to_string(const Expr& e) { return std::visit(/*...*/); }
int evaluate(const Expr& e) { return std::visit(/*...*/); }
// Easy to add operations; compiler checks exhaustiveness

// CRTP wins: Math library (zero-overhead, hot inner loop)
template<typename D> class VecBase {
public:
    auto dot(const D& o) const {
        const auto& s = static_cast<const D&>(*this);
        return s.x()*o.x() + s.y()*o.y() + s.z()*o.z();  // Inlined!
    }
};
// Millions of calls per frame — no vtable indirection

```

### Q3: Show a hybrid variant + virtual approach

**Answer:**

```cpp

#include <variant>
#include <memory>
#include <vector>
#include <iostream>

// Known types get fast variant dispatch
struct Circle { double r; };
struct Rect { double w, h; };

// Unknown types use virtual (escape hatch)
class CustomShape {
public:
    virtual ~CustomShape() = default;
    virtual double area() const = 0;
};

using Shape = std::variant<Circle, Rect, std::unique_ptr<CustomShape>>;

double area(const Shape& s) {
    return std::visit([](const auto& shape) -> double {
        using T = std::decay_t<decltype(shape)>;
        if constexpr (std::is_same_v<T, Circle>)
            return M_PI * shape.r * shape.r;
        else if constexpr (std::is_same_v<T, Rect>)
            return shape.w * shape.h;
        else
            return shape->area();  // Virtual only for custom types
    }, s);
}
// Best of both: fast for known types, extensible for custom

```

---

## Notes

- **Default choice for most code: virtual functions.** They're well-understood and flexible
- Use **variant** for closed sets (AST, message types, state machines) with value semantics
- Use **CRTP** only for measurably hot paths where vtable overhead matters
- Variant visitors get compile-time exhaustiveness checking — forgetting a case is a compile error
- The **Expression Problem**: virtual = easy to add types, hard to add operations; variant = opposite
