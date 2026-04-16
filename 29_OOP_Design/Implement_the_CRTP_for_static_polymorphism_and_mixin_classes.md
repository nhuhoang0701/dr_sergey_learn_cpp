# Implement the CRTP for static polymorphism and mixin classes

**Category:** OOP Design

---

## Topic Overview

**CRTP (Curiously Recurring Template Pattern):** A class `Derived` inherits from `Base<Derived>`. The base class can call derived class methods at compile time — achieving polymorphism with zero virtual dispatch overhead.

```cpp

template<typename Derived>
class Base {
    void interface() {
        static_cast<Derived*>(this)->implementation();  // Compile-time dispatch
    }
};
class Concrete : public Base<Concrete> {
    void implementation() { /* ... */ }
};

```

### CRTP Use Cases

| Use Case | What CRTP provides |
| --- | --- |
| Static polymorphism | Virtual-like dispatch at zero cost |
| Mixin classes | Add functionality via base class |
| Expression templates | Build expression trees at compile time |
| Operator generation | Auto-generate `!=`, `>`, `<=` from `==` and `<` |
| Singleton | Templated singleton base |

---

## Self-Assessment

### Q1: Implement CRTP for zero-overhead static polymorphism

**Answer:**

```cpp

#include <iostream>
#include <cmath>
#include <vector>
#include <chrono>

// ═══════════ CRTP base: static interface ═══════════
template<typename Derived>
class Shape {
public:
    double area() const {
        return static_cast<const Derived*>(this)->area_impl();
    }

    void draw() const {
        std::cout << "Drawing " << static_cast<const Derived*>(this)->name()
                  << " (area=" << area() << ")\n";
        static_cast<const Derived*>(this)->draw_impl();
    }

    // Mixin: add functionality to all shapes for free
    double perimeter_to_area_ratio() const {
        return static_cast<const Derived*>(this)->perimeter_impl() / area();
    }
};

class Circle : public Shape<Circle> {
    double r_;
    friend class Shape<Circle>;

    double area_impl() const { return M_PI * r_ * r_; }
    double perimeter_impl() const { return 2 * M_PI * r_; }
    void draw_impl() const { std::cout << "  ○ radius=" << r_ << "\n"; }
    const char* name() const { return "Circle"; }

public:
    explicit Circle(double r) : r_(r) {}
};

class Rectangle : public Shape<Rectangle> {
    double w_, h_;
    friend class Shape<Rectangle>;

    double area_impl() const { return w_ * h_; }
    double perimeter_impl() const { return 2 * (w_ + h_); }
    void draw_impl() const { std::cout << "  □ " << w_ << "×" << h_ << "\n"; }
    const char* name() const { return "Rectangle"; }

public:
    Rectangle(double w, double h) : w_(w), h_(h) {}
};

// Compile-time polymorphism — no vtable, no indirection
template<typename ShapeT>
void process_shape(const Shape<ShapeT>& shape) {
    shape.draw();  // Statically dispatched
    std::cout << "  P/A ratio: " << shape.perimeter_to_area_ratio() << "\n";
}

int main() {
    Circle c(5.0);
    Rectangle r(3.0, 4.0);

    process_shape(c);   // Compiles to direct call Circle::area_impl()
    process_shape(r);   // Compiles to direct call Rectangle::area_impl()
    // Zero overhead — no vtable lookup at runtime
    return 0;
}

```

### Q2: Use CRTP as mixin classes to add reusable functionality

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <type_traits>
#include <compare>

// ═══════════ CRTP Mixin 1: Equality operators ═══════════
template<typename Derived>
class EqualityComparable {
public:
    friend bool operator!=(const Derived& a, const Derived& b) {
        return !(a == b);  // Derived must provide operator==
    }
};

// ═══════════ CRTP Mixin 2: Printable ═══════════
template<typename Derived>
class Printable {
public:
    friend std::ostream& operator<<(std::ostream& os, const Derived& d) {
        d.print(os);       // Derived must provide print()
        return os;
    }

    std::string to_string() const {
        std::ostringstream oss;
        static_cast<const Derived*>(this)->print(oss);
        return oss.str();
    }
};

// ═══════════ CRTP Mixin 3: Clonable ═══════════
template<typename Derived>
class Clonable {
public:
    std::unique_ptr<Derived> clone() const {
        return std::make_unique<Derived>(static_cast<const Derived&>(*this));
    }
};

// ═══════════ CRTP Mixin 4: Registry (counts instances) ═══════════
template<typename Derived>
class InstanceCounter {
    static inline int count_ = 0;
public:
    InstanceCounter() { ++count_; }
    InstanceCounter(const InstanceCounter&) { ++count_; }
    ~InstanceCounter() { --count_; }
    static int instance_count() { return count_; }
};

// ═══════════ Compose multiple mixins ═══════════
class Color : public EqualityComparable<Color>,
              public Printable<Color>,
              public Clonable<Color>,
              public InstanceCounter<Color> {
    uint8_t r_, g_, b_;

public:
    Color(uint8_t r, uint8_t g, uint8_t b) : r_(r), g_(g), b_(b) {}

    bool operator==(const Color& o) const {
        return r_ == o.r_ && g_ == o.g_ && b_ == o.b_;
    }

    void print(std::ostream& os) const {
        os << "RGB(" << (int)r_ << "," << (int)g_ << "," << (int)b_ << ")";
    }
};

int main() {
    Color red(255, 0, 0);
    Color blue(0, 0, 255);

    std::cout << red << "\n";           // RGB(255,0,0) — via Printable mixin
    std::cout << (red != blue) << "\n"; // 1 — via EqualityComparable mixin
    auto red2 = red.clone();            // unique_ptr<Color> — via Clonable mixin
    std::cout << Color::instance_count() << "\n";  // 3 — via InstanceCounter

    return 0;
}

```

### Q3: Show the CRTP vs virtual dispatch performance difference and C++23 deducing this

**Answer:**

```cpp

// ═══════════ C++23: Deducing this replaces CRTP ═══════════
// Before (CRTP):
template<typename Derived>
class OldPrintable {
public:
    void print() const {
        static_cast<const Derived*>(this)->do_print();
    }
};

class OldWidget : public OldPrintable<OldWidget> {
public:
    void do_print() const { std::cout << "Widget\n"; }
};

// After (C++23 deducing this — no CRTP needed!):
class NewPrintable {
public:
    void print(this const auto& self) {  // Deducing this
        self.do_print();                  // Calls derived method directly
    }
};

class NewWidget : public NewPrintable {
public:
    void do_print() const { std::cout << "Widget\n"; }
};
// No template parameter on the base class — much cleaner!

// ═══════════ Performance comparison ═══════════
/*
Benchmark: 10M dispatch calls on x86_64

Virtual dispatch:

  - Indirect call through vtable pointer
  - ~2-5ns per call (cache miss: ~50ns)
  - Cannot be inlined by compiler

CRTP dispatch:

  - Direct function call (resolved at compile time)
  - ~0.3ns per call (always inlinable)
  - Compiler can optimize across call boundary

Typical results:
  Virtual:    35ms for 10M calls
  CRTP:       3ms for 10M calls  (10-12x faster)

When the difference matters:

  - Inner loops (physics sim, rendering, signal processing)
  - Embedded real-time (ISR, control loops)
  - Hot paths in high-frequency trading

When it DOESN'T matter:

  - Infrequent calls (UI buttons, config parsing)
  - IO-bound code (file, network)
  - Code where polymorphism is runtime-determined

*/

```

---

## Notes

- CRTP has two costs: longer compile times and complex error messages
- C++23 `deducing this` eliminates most CRTP use cases — prefer it when available
- CRTP objects **cannot** be stored in a single container (unlike virtual polymorphism) — each `Shape<T>` is a different type
- Use CRTP for tight loops, embedded, and performance-critical code; use virtual for extensible plugins and heterogeneous collections
- The "friend class" declaration in derived classes is needed to grant CRTP base access to private impl methods
