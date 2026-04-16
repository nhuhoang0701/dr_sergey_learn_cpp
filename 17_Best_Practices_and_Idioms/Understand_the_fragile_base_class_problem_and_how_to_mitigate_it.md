# Understand the fragile base class problem and how to mitigate it

**Category:** Best Practices & Idioms  
**Item:** #797  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rh-copy>  

---

## Topic Overview

The **fragile base class problem** means that changes to a base class can break derived classes in subtle, unpredictable ways — even without changing the derived class source code.

### How It Happens

```cpp

Base class author adds/changes a method
        |
        v
Derived class silently inherits the change
        |
        v
Derived class behavior breaks!
(name hiding, changed semantics, ABI break)

```

### Common Manifestations

| Change in Base | Problem in Derived |
| --- | --- |
| Add method `foo(int)` | Hides derived's `foo(string)` (name hiding) |
| Change virtual method semantics | Derived override now violates new contract |
| Add data member | Changes `sizeof(Base)`, breaks ABI |
| Reorder virtual methods | Breaks vtable layout |
| Add default parameter | Derived inherits old default at call site |

---

## Self-Assessment

### Q1: Show how adding a non-virtual method to a base class breaks derived classes

```cpp

#include <iostream>
#include <string>

// Version 1: Base has no foo(int)
class BaseV1 {
public:
    virtual ~BaseV1() = default;
    void greet() { std::cout << "Hello from Base\n"; }
};

class DerivedV1 : public BaseV1 {
public:
    void foo(const std::string& s) {
        std::cout << "Derived::foo(string): " << s << '\n';
    }
};

// Version 2: Base author adds foo(int) — BREAKS Derived!
class BaseV2 {
public:
    virtual ~BaseV2() = default;
    void greet() { std::cout << "Hello from Base\n"; }
    void foo(int x) {  // NEW! Added in V2
        std::cout << "Base::foo(int): " << x << '\n';
    }
};

class DerivedV2 : public BaseV2 {
public:
    void foo(const std::string& s) {
        // This HIDES Base::foo(int)!
        // But if someone calls derived.foo(42), it may
        // silently convert 42 to string or fail to compile
        std::cout << "Derived::foo(string): " << s << '\n';
    }
};

int main() {
    DerivedV2 d;
    d.foo("hello");   // OK: calls Derived::foo(string)
    // d.foo(42);      // ERROR: Base::foo(int) is HIDDEN!
    d.BaseV2::foo(42); // Must qualify to reach Base version

    // The derived class author didn't change anything,
    // but adding foo(int) to Base changed behavior!
    std::cout << "Base method hidden by derived!\n";
}
// Expected output:
// Derived::foo(string): hello
// Base::foo(int): 42
// Base method hidden by derived!

```

### Q2: Show how the NVI (Non-Virtual Interface) pattern reduces the fragile base class problem

```cpp

#include <iostream>
#include <string>

// NVI Pattern: public non-virtual interface, private virtual implementation
class Shape {
public:
    // Non-virtual public interface — STABLE, never changes
    void draw() const {
        pre_draw();      // invariant: setup
        do_draw();       // customization point
        post_draw();     // invariant: cleanup
    }

    double area() const {
        return do_area();  // delegates to private virtual
    }

    virtual ~Shape() = default;

private:
    // Private virtual — derived classes customize ONLY these
    virtual void do_draw() const = 0;
    virtual double do_area() const = 0;

    // Non-virtual helpers — base controls the framework
    void pre_draw() const { std::cout << "[begin draw] "; }
    void post_draw() const { std::cout << " [end draw]\n"; }
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : radius_(r) {}
private:
    void do_draw() const override {
        std::cout << "Circle(r=" << radius_ << ")";
    }
    double do_area() const override {
        return 3.14159 * radius_ * radius_;
    }
};

class Square : public Shape {
    double side_;
public:
    Square(double s) : side_(s) {}
private:
    void do_draw() const override {
        std::cout << "Square(s=" << side_ << ")";
    }
    double do_area() const override { return side_ * side_; }
};

int main() {
    Circle c(5.0);
    Square s(4.0);

    c.draw();  // base controls pre/post, derived controls do_draw()
    s.draw();
    std::cout << "Circle area: " << c.area() << '\n';
    std::cout << "Square area: " << s.area() << '\n';
}
// Expected output:
// [begin draw] Circle(r=5) [end draw]
// [begin draw] Square(s=4) [end draw]
// Circle area: 78.5398
// Square area: 16

```

**Why NVI helps:** Base class author can add pre/post steps, logging, validation WITHOUT affecting any derived class.

### Q3: Use composition to eliminate the fragile base class problem

```cpp

#include <functional>
#include <iostream>
#include <string>

// Instead of inheritance, compose behaviors
struct DrawBehavior {
    virtual void draw() const = 0;
    virtual ~DrawBehavior() = default;
};

class CircleDraw : public DrawBehavior {
    double radius_;
public:
    CircleDraw(double r) : radius_(r) {}
    void draw() const override {
        std::cout << "Drawing circle r=" << radius_ << '\n';
    }
};

class SquareDraw : public DrawBehavior {
    double side_;
public:
    SquareDraw(double s) : side_(s) {}
    void draw() const override {
        std::cout << "Drawing square s=" << side_ << '\n';
    }
};

// Widget COMPOSES drawing behavior — no fragile base class!
class Widget {
    std::string name_;
    std::unique_ptr<DrawBehavior> drawer_;  // has-a, not is-a
public:
    Widget(std::string name, std::unique_ptr<DrawBehavior> drawer)
        : name_(std::move(name)), drawer_(std::move(drawer)) {}

    void render() const {
        std::cout << "Widget '" << name_ << "': ";
        drawer_->draw();
    }

    // Can swap behavior at runtime!
    void set_drawer(std::unique_ptr<DrawBehavior> d) {
        drawer_ = std::move(d);
    }
};

int main() {
    Widget w("MyWidget", std::make_unique<CircleDraw>(3.0));
    w.render();

    w.set_drawer(std::make_unique<SquareDraw>(5.0));
    w.render();
}
// Expected output:
// Widget 'MyWidget': Drawing circle r=3
// Widget 'MyWidget': Drawing square s=5

```

---

## Notes

- Mark base classes as `final` if they shouldn't be inherited from.
- Use `override` on every virtual override to catch base class changes at compile time.
- NVI is the C++ Core Guidelines' recommended approach (C.133).
- The fragile base class problem is a key reason modern C++ favors composition and templates over deep inheritance hierarchies.
