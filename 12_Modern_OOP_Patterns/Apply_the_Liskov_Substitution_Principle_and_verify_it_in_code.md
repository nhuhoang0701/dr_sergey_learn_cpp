# Apply the Liskov Substitution Principle and verify it in code

**Category:** Modern OOP Patterns  
**Item:** #385  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rh-copy>  

---

## Topic Overview

The **Liskov Substitution Principle (LSP)** states: *If S is a subtype of T, then objects of type T may be replaced with objects of type S without altering program correctness.* In practical C++ terms: any code that works with a `Base&` must also work correctly when handed a `Derived&`.

### LSP Rules (Behavioral Subtyping)

| Rule | Meaning | Violation Example |
| --- | --- | --- |
| **Preconditions cannot be strengthened** | Derived can't require MORE from callers | `Base::deposit(amount > 0)` → `Derived::deposit(amount > 100)` ❌ |
| **Postconditions cannot be weakened** | Derived can't promise LESS to callers | `Base::sort()` → always sorted. `Derived::sort()` → sometimes not ❌ |
| **Invariants must be preserved** | Derived can't break base class invariants | Rectangle's area invariant broken by Square ❌ |
| **History constraint** | Derived can't allow state changes the base forbids | Immutable base → mutable derived ❌ |

### Visual Check

```cpp

Function f(Base& b):
  b.set_width(5);
  b.set_height(3);
  assert(b.area() == 15);  // MUST hold for ALL subtypes
  
  If Square : Base → set_width(5) also sets height=5 → area=25 ≠ 15 → LSP VIOLATION

```

---

## Self-Assessment

### Q1: Give a canonical example where derived class preconditions are stronger than base (LSP violation)

**Solution — Strengthened Preconditions:**

```cpp

#include <iostream>
#include <stdexcept>
#include <cassert>
#include <memory>

// Base class: accepts any positive amount
class Account {
protected:
    double balance_ = 0.0;
public:
    virtual ~Account() = default;

    // Precondition: amount > 0
    virtual void deposit(double amount) {
        if (amount <= 0)
            throw std::invalid_argument("Amount must be positive");
        balance_ += amount;
    }

    double balance() const { return balance_; }
};

// ❌ LSP VIOLATION: strengthened precondition
class PremiumAccount : public Account {
public:
    // Precondition: amount >= 1000 (STRONGER than base's amount > 0)
    void deposit(double amount) override {
        if (amount < 1000)
            throw std::invalid_argument("Premium accounts require min $1000 deposit");
        balance_ += amount * 1.05;  // 5% bonus
    }
};

// Function that works with base class — expects base's precondition
void make_small_deposit(Account& acct) {
    acct.deposit(50.0);  // Valid per Base's contract (50 > 0)
    std::cout << "Balance: " << acct.balance() << "\n";
}

int main() {
    Account regular;
    make_small_deposit(regular);  // ✅ Works: 50 > 0

    PremiumAccount premium;
    try {
        make_small_deposit(premium);  // ❌ THROWS: 50 < 1000
    } catch (const std::exception& e) {
        std::cout << "LSP Violation: " << e.what() << "\n";
    }

    // === FIXED VERSION ===
    // PremiumAccount should accept amount > 0 (same as or weaker than base)
    // and add its bonus logic internally:
    // void deposit(double amount) override {
    //     Account::deposit(amount);       // honor base precondition
    //     if (amount >= 1000)
    //         balance_ += amount * 0.05;  // bonus on top
    // }
}
// Expected output:
//   Balance: 50
//   LSP Violation: Premium accounts require min $1000 deposit

```

---

### Q2: Show how a Rectangle/Square hierarchy violates LSP and refactor it to satisfy it

**Solution — The Classic Rectangle/Square Problem:**

```cpp

#include <iostream>
#include <memory>
#include <cassert>

// ❌ LSP VIOLATION: Square derives from Rectangle
class Rectangle {
protected:
    int width_, height_;
public:
    Rectangle(int w, int h) : width_(w), height_(h) {}
    virtual ~Rectangle() = default;

    virtual void set_width(int w) { width_ = w; }
    virtual void set_height(int h) { height_ = h; }
    int width() const { return width_; }
    int height() const { return height_; }
    int area() const { return width_ * height_; }
};

class Square : public Rectangle {
public:
    Square(int side) : Rectangle(side, side) {}

    // Must maintain w == h invariant → overrides set_width/set_height
    void set_width(int w) override { width_ = w; height_ = w; }
    void set_height(int h) override { width_ = h; height_ = h; }
};

// Function that expects Rectangle behavior
void test_rectangle(Rectangle& r) {
    r.set_width(5);
    r.set_height(3);
    // Rectangle contract: width * height = area
    int expected_area = 5 * 3;  // = 15
    std::cout << "Expected: " << expected_area
              << ", Got: " << r.area() << "\n";
    assert(r.area() == expected_area);  // FAILS for Square!
}

int main() {
    Rectangle rect(4, 4);
    test_rectangle(rect);  // ✅ OK: 5 * 3 = 15

    Square sq(4);
    try {
        test_rectangle(sq);  // ❌ FAILS: set_height(3) → 3*3 = 9, not 15
    } catch (...) {}
}
// Expected output:
//   Expected: 15, Got: 15
//   Expected: 15, Got: 9
//   (assertion failure)

```

**Refactored — LSP-Compliant Design:**

```cpp

#include <iostream>
#include <memory>
#include <cmath>

// ✅ SOLUTION: Shape is the interface, Rectangle and Square are siblings

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual double perimeter() const = 0;
    virtual void scale(double factor) = 0;
};

class Rectangle : public Shape {
    double width_, height_;
public:
    Rectangle(double w, double h) : width_(w), height_(h) {}
    double area() const override { return width_ * height_; }
    double perimeter() const override { return 2 * (width_ + height_); }
    void scale(double f) override { width_ *= f; height_ *= f; }
    double width() const { return width_; }
    double height() const { return height_; }
};

class Square : public Shape {
    double side_;
public:
    Square(double s) : side_(s) {}
    double area() const override { return side_ * side_; }
    double perimeter() const override { return 4 * side_; }
    void scale(double f) override { side_ *= f; }
    double side() const { return side_; }
};

// Now code using Shape& works correctly with both
void describe(const Shape& s) {
    std::cout << "  Area: " << s.area()
              << ", Perimeter: " << s.perimeter() << "\n";
}

int main() {
    Rectangle r(5, 3);
    Square s(4);

    std::cout << "Rectangle:\n"; describe(r);
    std::cout << "Square:\n";    describe(s);

    r.scale(2);
    s.scale(2);

    std::cout << "After scale(2):\n";
    std::cout << "Rectangle:\n"; describe(r);
    std::cout << "Square:\n";    describe(s);
}
// Expected output:
//   Rectangle:
//     Area: 15, Perimeter: 16
//   Square:
//     Area: 16, Perimeter: 16
//   After scale(2):
//   Rectangle:
//     Area: 60, Perimeter: 32
//   Square:
//     Area: 64, Perimeter: 32

```

---

### Q3: Explain how concepts can encode LSP requirements at compile time for generic code

**Solution — Concepts as Compile-Time LSP Enforcement:**

```cpp

#include <iostream>
#include <concepts>
#include <string>
#include <vector>
#include <memory>

// Concept defines the "behavioral contract" — the LSP requirements
template <typename T>
concept Drawable = requires(T t, int x, int y) {
    { t.draw() } -> std::same_as<void>;
    { t.area() } -> std::convertible_to<double>;
    { t.name() } -> std::convertible_to<std::string>;
};

// Concept for containers — models the container LSP contract
template <typename C>
concept Container = requires(C c, typename C::value_type v) {
    { c.size() } -> std::convertible_to<std::size_t>;
    { c.empty() } -> std::same_as<bool>;
    { c.push_back(v) };
    // Any type satisfying Container can substitute for another
    // without breaking code that uses these operations
};

// Generic code constrained by concept — LSP at compile time
void render(const Drawable auto& shape) {
    std::cout << shape.name() << ": area = " << shape.area() << "\n";
    shape.draw();
}

// Types that satisfy the concept (behavioral subtypes)
struct Circle {
    double radius;
    void draw() const { std::cout << "  Drawing circle (r=" << radius << ")\n"; }
    double area() const { return 3.14159 * radius * radius; }
    std::string name() const { return "Circle"; }
};

struct Triangle {
    double base, height;
    void draw() const { std::cout << "  Drawing triangle\n"; }
    double area() const { return 0.5 * base * height; }
    std::string name() const { return "Triangle"; }
};

// ❌ This type does NOT satisfy the concept — compile error if used
struct BadShape {
    void draw() { /* non-const! */ }
    // missing area() and name()
};

int main() {
    Circle c{5.0};
    Triangle t{10.0, 4.0};

    render(c);  // ✅ Circle satisfies Drawable
    render(t);  // ✅ Triangle satisfies Drawable

    // render(BadShape{});  // ❌ compile error: doesn't satisfy Drawable

    // Verify concept satisfaction at compile time:
    static_assert(Drawable<Circle>);
    static_assert(Drawable<Triangle>);
    static_assert(!Drawable<BadShape>);
    std::cout << "All concept checks passed.\n";
}
// Expected output:
//   Circle: area = 78.5398
//     Drawing circle (r=5)
//   Triangle: area = 20
//     Drawing triangle
//   All concept checks passed.

```

**Concepts as Compile-Time LSP:**

```cpp

Traditional LSP (runtime):
  Code using Base& works for any Derived&
  Violation detected at RUNTIME (unexpected behavior)

Concept-based LSP (compile-time):
  Code using Drawable auto works for any type satisfying Drawable
  Violation detected at COMPILE TIME (clear error message)

  Stronger guarantee: if it compiles, the behavioral contract is met

```

---

## Notes

- **LSP is about behavior, not just interface** — a class can implement all methods of an interface but still violate LSP if postconditions differ.
- **Covariant return types** are LSP-compatible — returning a more specific type than the base is fine.
- **`final`** can help enforce LSP — prevent further derivation if subclassing would risk violation.
- **"Prefer composition over inheritance"** often avoids LSP issues entirely — if Square doesn't inherit from Rectangle, there's no substitution problem.
- **Concepts (C++20)** provide compile-time LSP for generic code — types either satisfy the concept or produce a clear compiler error.
- **Design smell:** If a derived class throws `UnsupportedOperationException` or does nothing in an override, it's likely violating LSP.
