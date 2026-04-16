# Design type-erased interfaces for runtime polymorphism without inheritance

**Category:** OOP Design

---

## Topic Overview

**Type erasure** provides runtime polymorphism (like virtual functions) WITHOUT requiring types to inherit from a common base class. The classic example is `std::function<void()>` — it can hold a lambda, function pointer, or functor without any of them inheriting from anything.

```cpp

Virtual polymorphism:         Type erasure:
  IDrawable* p;                 Drawable d;
  p = new Circle();             d = Circle{5};     // No inheritance needed
  p = new Rect();               d = Rect{3, 4};    // Circle/Rect are independent
  p->draw();                    d.draw();           // Same runtime dispatch

```

### Type Erasure Pattern

```cpp

┌─────────────────────────────────────────────┐
│ Drawable (value type, owns concept*)        │
│ ┌─────────────────────────────────────────┐ │
│ │ concept (ABC)                           │ │
│ │   virtual draw() = 0                    │ │
│ │   virtual clone() = 0                   │ │
│ ├─────────────────────────────────────────┤ │
│ │ model<T> : concept                      │ │
│ │   T obj_;                               │ │
│ │   draw() { obj_.draw(); }               │ │
│ │   clone() { return new model(obj_); }   │ │
│ └─────────────────────────────────────────┘ │
└─────────────────────────────────────────────┘

```

---

## Self-Assessment

### Q1: Implement a type-erased Drawable using the concept/model pattern

**Answer:**

```cpp

#include <memory>
#include <vector>
#include <iostream>
#include <cmath>

// ═══════════ Type-erased Drawable ═══════════
class Drawable {
    // INTERNAL: abstract concept
    struct Concept {
        virtual ~Concept() = default;
        virtual void draw(std::ostream& os) const = 0;
        virtual double area() const = 0;
        virtual std::unique_ptr<Concept> clone() const = 0;
    };

    // INTERNAL: concrete model (wraps any type with draw() and area())
    template<typename T>
    struct Model final : Concept {
        T obj_;
        explicit Model(T obj) : obj_(std::move(obj)) {}
        void draw(std::ostream& os) const override { obj_.draw(os); }
        double area() const override { return obj_.area(); }
        std::unique_ptr<Concept> clone() const override {
            return std::make_unique<Model>(obj_);
        }
    };

    std::unique_ptr<Concept> pimpl_;

public:
    // Accept ANY type that has draw() and area() — no inheritance required
    template<typename T>
    Drawable(T obj) : pimpl_(std::make_unique<Model<T>>(std::move(obj))) {}

    // Value semantics: copyable!
    Drawable(const Drawable& other) : pimpl_(other.pimpl_->clone()) {}
    Drawable& operator=(const Drawable& other) {
        pimpl_ = other.pimpl_->clone();
        return *this;
    }
    Drawable(Drawable&&) noexcept = default;
    Drawable& operator=(Drawable&&) noexcept = default;

    // Public interface
    void draw(std::ostream& os) const { pimpl_->draw(os); }
    double area() const { return pimpl_->area(); }
};

// ═══════════ Concrete types — NO base class, NO virtual ═══════════
struct Circle {
    double radius;
    void draw(std::ostream& os) const {
        os << "Circle(r=" << radius << ")\n";
    }
    double area() const { return M_PI * radius * radius; }
};

struct Square {
    double side;
    void draw(std::ostream& os) const {
        os << "Square(s=" << side << ")\n";
    }
    double area() const { return side * side; }
};

struct Triangle {
    double base, height;
    void draw(std::ostream& os) const {
        os << "Triangle(b=" << base << ", h=" << height << ")\n";
    }
    double area() const { return 0.5 * base * height; }
};

int main() {
    // Heterogeneous container — no inheritance needed!
    std::vector<Drawable> shapes;
    shapes.push_back(Circle{5.0});
    shapes.push_back(Square{3.0});
    shapes.push_back(Triangle{4.0, 6.0});

    for (const auto& s : shapes) {
        s.draw(std::cout);
        std::cout << "  area = " << s.area() << "\n";
    }

    // Value semantics: can copy!
    Drawable a = Circle{10.0};
    Drawable b = a;  // Deep copy
    return 0;
}

```

### Q2: Compare type erasure with virtual inheritance and std::variant

**Answer:**

| Feature | Virtual Inheritance | Type Erasure | std::variant |
| --- | :---: | :---: | :---: |
| Open set (add new types) | **Yes** | **Yes** | No (closed set) |
| Heap allocation | Yes (polymorphic) | Yes (small buffer opt. possible) | **No** |
| Value semantics | No (slicing problem) | **Yes** (copyable) | **Yes** |
| Compile-time checked | No | No | **Yes** (exhaustive visit) |
| Types must inherit base | **Yes** | **No** | **No** |
| Container storage | `vector<unique_ptr<Base>>` | `vector<Drawable>` | `vector<variant<A,B,C>>` |
| Adding new operations | Requires base change | Requires wrapper change | **Easy** (new visitor) |
| Cache locality | Poor (pointer chasing) | Better (SBO possible) | **Best** (inline) |

```cpp

// Virtual: types must inherit, stored via pointer
std::vector<std::unique_ptr<IShape>> v1;
v1.push_back(std::make_unique<Circle>(5));

// Type erasure: types are independent, stored by value
std::vector<Drawable> v2;
v2.push_back(Circle{5});           // No make_unique, value semantics

// Variant: closed set, best performance
using Shape = std::variant<Circle, Square, Triangle>;
std::vector<Shape> v3;
v3.push_back(Circle{5});           // Inline storage, no heap

```

### Q3: Implement type erasure with Small Buffer Optimization (SBO)

**Answer:**

```cpp

#include <cstddef>
#include <new>
#include <type_traits>
#include <utility>
#include <iostream>

// ═══════════ Type-erased callable with SBO (like std::function) ═══════════
template<typename Signature>
class Function;

template<typename R, typename... Args>
class Function<R(Args...)> {
    struct Concept {
        virtual ~Concept() = default;
        virtual R invoke(Args... args) = 0;
        virtual void clone_to(void* dest) const = 0;
        virtual size_t size() const = 0;
    };

    template<typename F>
    struct Model : Concept {
        F func_;
        explicit Model(F f) : func_(std::move(f)) {}
        R invoke(Args... args) override {
            return func_(std::forward<Args>(args)...);
        }
        void clone_to(void* dest) const override {
            new (dest) Model(func_);
        }
        size_t size() const override { return sizeof(Model); }
    };

    // Small buffer: avoid heap for small callables (lambdas, function pointers)
    static constexpr size_t SBO_SIZE = 48;
    alignas(std::max_align_t) char buffer_[SBO_SIZE];
    Concept* ptr_ = nullptr;
    bool on_heap_ = false;

    template<typename F>
    void construct(F&& f) {
        using ModelT = Model<std::decay_t<F>>;
        if constexpr (sizeof(ModelT) <= SBO_SIZE &&
                      std::is_nothrow_move_constructible_v<std::decay_t<F>>) {
            // SBO: construct in-place
            ptr_ = new (buffer_) ModelT(std::forward<F>(f));
            on_heap_ = false;
        } else {
            // Too large: heap allocate
            ptr_ = new ModelT(std::forward<F>(f));
            on_heap_ = true;
        }
    }

    void destroy() {
        if (ptr_) {
            if (on_heap_) delete ptr_;
            else ptr_->~Concept();  // Destroy in-place
            ptr_ = nullptr;
        }
    }

public:
    Function() = default;

    template<typename F>
    Function(F&& f) { construct(std::forward<F>(f)); }

    ~Function() { destroy(); }

    Function(const Function& other) {
        if (other.ptr_) {
            if (!other.on_heap_ && other.ptr_->size() <= SBO_SIZE) {
                other.ptr_->clone_to(buffer_);
                ptr_ = reinterpret_cast<Concept*>(buffer_);
                on_heap_ = false;
            } else {
                other.ptr_->clone_to(nullptr); // simplified
                on_heap_ = true;
            }
        }
    }

    R operator()(Args... args) {
        if (!ptr_) throw std::bad_function_call();
        return ptr_->invoke(std::forward<Args>(args)...);
    }

    explicit operator bool() const { return ptr_ != nullptr; }
};

int main() {
    // Small lambda: stored in SBO buffer (no heap allocation!)
    Function<int(int, int)> add = [](int a, int b) { return a + b; };
    std::cout << add(3, 4) << "\n";  // 7

    // Lambda with captures: may use SBO if small enough
    int multiplier = 10;
    Function<int(int)> mul = [multiplier](int x) { return x * multiplier; };
    std::cout << mul(5) << "\n";     // 50
    return 0;
}

```

---

## Notes

- `std::function`, `std::any`, `std::move_only_function` are all type-erased types in the standard
- Type erasure = virtual dispatch hidden inside a value-type wrapper
- SBO avoids heap allocation for small objects (typically 16-64 bytes inline buffer)
- Use type erasure when: types are independent (can't add base class), but you need heterogeneous collections
- Type erasure has the same overhead as virtual dispatch (indirection) — it's NOT faster, just more flexible
- Sean Parent's "Runtime Polymorphism" talk is the canonical reference for this pattern
