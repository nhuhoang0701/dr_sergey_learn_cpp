# Understand object slicing and how to prevent it

**Category:** Modern OOP Patterns  
**Item:** #104  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/derived_class>  

---

## Topic Overview

**Object slicing** occurs when a derived class object is copied into a base class variable *by value*. The derived part is "sliced off" — only the base class portion is copied. This silently loses data and polymorphic behavior.

### Slicing Visualized

```cpp

Derived object in memory:
┌────────────────────────────────┐
│ Base members:                  │
│   name_ = "Rex"               │
│   vptr → Dog vtable           │
├────────────────────────────────┤
│ Derived members:               │
│   breed_ = "Labrador"         │  ← THIS IS LOST
│   tricks_ = 5                 │  ← THIS IS LOST
└────────────────────────────────┘

After slicing (Base b = derived_obj):
┌────────────────────────────────┐
│ Base members:                  │
│   name_ = "Rex"               │
│   vptr → Animal vtable  ←!    │  ← Vtable changed to Base!
└────────────────────────────────┘
   Derived data is GONE.
   Virtual calls go to Base, not Derived.

```

---

## Self-Assessment

### Q1: Demonstrate object slicing when a derived object is passed by value as a base type

**Solution:**

```cpp

#include <iostream>
#include <string>
#include <vector>

class Animal {
    std::string name_;
public:
    explicit Animal(std::string n) : name_(std::move(n)) {}
    virtual ~Animal() = default;

    virtual void speak() const {
        std::cout << "  " << name_ << ": [generic animal sound]\n";
    }

    const std::string& name() const { return name_; }
};

class Dog : public Animal {
    std::string breed_;
public:
    Dog(std::string n, std::string b)
        : Animal(std::move(n)), breed_(std::move(b)) {}

    void speak() const override {
        std::cout << "  " << name() << " (" << breed_ << "): Woof!\n";
    }
};

// ❌ Takes Animal BY VALUE → slicing!
void make_speak_sliced(Animal a) {
    a.speak();  // Calls Animal::speak(), not Dog::speak()!
}

// ✅ Takes Animal by REFERENCE → no slicing
void make_speak_safe(const Animal& a) {
    a.speak();  // Calls the actual derived type's speak()
}

int main() {
    Dog rex("Rex", "Labrador");

    std::cout << "Direct call:\n";
    rex.speak();  // Dog::speak()

    std::cout << "\nBy value (SLICED!):\n";
    make_speak_sliced(rex);  // Animal::speak() — breed_ lost!

    std::cout << "\nBy reference (correct):\n";
    make_speak_safe(rex);  // Dog::speak() — polymorphism preserved

    // Slicing in containers:
    std::cout << "\nSlicing in vector<Animal>:\n";
    std::vector<Animal> animals;
    animals.push_back(rex);       // ❌ SLICED! Push copies the Base portion only
    animals[0].speak();           // Animal::speak()

    std::cout << "\nNo slicing in vector<Animal*>:\n";
    std::vector<const Animal*> ptrs;
    ptrs.push_back(&rex);        // ✅ Pointer, no copy
    ptrs[0]->speak();            // Dog::speak()
}
// Expected output:
//   Direct call:
//     Rex (Labrador): Woof!
//
//   By value (SLICED!):
//     Rex: [generic animal sound]     ← lost Dog behavior!
//
//   By reference (correct):
//     Rex (Labrador): Woof!
//
//   Slicing in vector<Animal>:
//     Rex: [generic animal sound]     ← lost Dog behavior!
//
//   No slicing in vector<Animal*>:
//     Rex (Labrador): Woof!

```

---

### Q2: Show how deleting the copy constructor of a base class prevents accidental slicing

**Solution:**

```cpp

#include <iostream>
#include <string>
#include <memory>

class Shape {
    std::string type_;
public:
    explicit Shape(std::string t) : type_(std::move(t)) {}
    virtual ~Shape() = default;

    // ❌ DELETE copy/move to prevent slicing entirely
    Shape(const Shape&) = delete;
    Shape& operator=(const Shape&) = delete;
    Shape(Shape&&) = delete;
    Shape& operator=(Shape&&) = delete;

    virtual double area() const = 0;
    const std::string& type() const { return type_; }
};

class Circle : public Shape {
    double radius_;
public:
    explicit Circle(double r) : Shape("Circle"), radius_(r) {}

    double area() const override { return 3.14159 * radius_ * radius_; }
};

// void takes_by_value(Shape s) { }     // ❌ COMPILE ERROR: deleted copy
// Shape s = Circle(5.0);                // ❌ COMPILE ERROR: deleted copy
// std::vector<Shape> shapes;            // ❌ COMPILE ERROR: requires copy

int main() {
    Circle c(5.0);

    // ✅ Reference: no copy needed
    const Shape& ref = c;
    std::cout << "Type: " << ref.type() << ", Area: " << ref.area() << "\n";

    // ✅ Pointer: no copy needed
    Shape* ptr = &c;
    std::cout << "Type: " << ptr->type() << ", Area: " << ptr->area() << "\n";

    // ✅ Smart pointer: owns polymorphic object
    auto sp = std::make_unique<Circle>(3.0);
    std::cout << "Type: " << sp->type() << ", Area: " << sp->area() << "\n";
}
// Expected output:
//   Type: Circle, Area: 78.5398
//   Type: Circle, Area: 78.5398
//   Type: Circle, Area: 28.2743

```

**Alternative: Protected copy constructor (allows derived copy but prevents base slicing):**

```cpp

class SafeBase {
protected:
    SafeBase(const SafeBase&) = default;  // derived can copy themselves
    SafeBase& operator=(const SafeBase&) = default;
public:
    SafeBase() = default;
    virtual ~SafeBase() = default;
};

class Derived : public SafeBase {
public:
    // Can copy Derived objects (copy ctor calls protected base copy)
    Derived(const Derived&) = default;
};

// SafeBase b1;
// SafeBase b2 = b1;   // ❌ COMPILE ERROR: copy is protected
// Derived d1;
// Derived d2 = d1;    // ✅ OK: derived copy works

```

---

### Q3: Explain why polymorphic types should not be copyable and how `unique_ptr` enforces this

**Why polymorphic types shouldn't be copyable:**

| Problem | Explanation |
| --- | --- |
| **Object slicing** | Copying `Derived` into `Base` loses derived data |
| **Vtable mismatch** | The copy's vtable points to `Base`, not `Derived` |
| **Semantic confusion** | What does "copy a polymorphic object" mean? The base class can't know the derived class's size or members |
| **Resource duplication** | Derived classes may hold unique resources (file handles, connections) that shouldn't be implicitly shared |

**`unique_ptr` as the solution:**

```cpp

#include <iostream>
#include <memory>
#include <string>
#include <vector>

class Widget {
public:
    virtual ~Widget() = default;
    Widget(const Widget&) = delete;             // ❌ No slicing
    Widget& operator=(const Widget&) = delete;
    Widget() = default;

    virtual void render() const = 0;

    // Explicit clone for intentional deep copy
    virtual std::unique_ptr<Widget> clone() const = 0;
};

class Button : public Widget {
    std::string label_;
public:
    explicit Button(std::string l) : label_(std::move(l)) {}
    void render() const override {
        std::cout << "  [Button: " << label_ << "]\n";
    }
    std::unique_ptr<Widget> clone() const override {
        return std::make_unique<Button>(label_);
    }
};

class Label : public Widget {
    std::string text_;
public:
    explicit Label(std::string t) : text_(std::move(t)) {}
    void render() const override {
        std::cout << "  [Label: " << text_ << "]\n";
    }
    std::unique_ptr<Widget> clone() const override {
        return std::make_unique<Label>(text_);
    }
};

int main() {
    // unique_ptr prevents slicing at EVERY level:
    std::vector<std::unique_ptr<Widget>> widgets;
    widgets.push_back(std::make_unique<Button>("OK"));
    widgets.push_back(std::make_unique<Label>("Hello"));

    // widgets.push_back(widgets[0]);     // ❌ COMPILE ERROR: unique_ptr not copyable
    // Widget w = *widgets[0];            // ❌ COMPILE ERROR: Widget not copyable

    // Explicit clone when you WANT a copy:
    widgets.push_back(widgets[0]->clone());

    std::cout << "Rendering all widgets:\n";
    for (const auto& w : widgets)
        w->render();
}
// Expected output:
//   Rendering all widgets:
//     [Button: OK]
//     [Label: Hello]
//     [Button: OK]     ← intentional clone

```

**How `unique_ptr` enforces non-slicing:**

```cpp

1. unique_ptr<Widget> is move-only — can't copy the container
2. Widget copy ctor is deleted — can't copy the object
3. The only way to "copy" is explicit clone() — intentional, correct
4. No accidental by-value passing: unique_ptr forces pointer semantics

Result: slicing is IMPOSSIBLE at compile time.

```

---

## Notes

- **Clang-Tidy** has `cppcoreguidelines-slicing` check that warns about copy/move of polymorphic types.
- **GCC `-Wextra`** does NOT warn about slicing — it's syntactically valid C++. Use static analysis tools.
- **`std::any` also slices internally** — `std::any a = derived;` stores a copy, which is a sliced copy if `derived` is passed as `Base`.
- **C++ Core Guidelines C.67:** "A polymorphic class should suppress public copy/move."
- **`std::polymorphic_value` (P0201)** is a proposed smart pointer that correctly deep-copies polymorphic objects (value semantics + no slicing). Not yet standardized.
