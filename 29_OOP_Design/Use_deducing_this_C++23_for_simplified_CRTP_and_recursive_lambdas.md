# Use deducing this (C++23) for simplified CRTP and recursive lambdas

**Category:** OOP Design

---

## Topic Overview

C++23's **deducing this** (`this auto&& self`) allows a member function to deduce its own type, eliminating the need for CRTP:

```cpp

Before (CRTP):   template<typename D> class Base { auto& self() { return static_cast<D&>(*this); } };
After (C++23):   class Base { void foo(this auto&& self) { self.bar(); } };

```

| Feature | CRTP | Deducing this |
| --- | :---: | :---: |
| Syntax complexity | High | **Simple** |
| Const/non-const overloads | 2 functions needed | **1 function** |
| Works with lambdas | No | **Yes** |
| Recursive lambdas | Requires Y-combinator | **Direct** |
| Move-from detection | Not possible | **Yes** (`self` is rvalue ref) |

---

## Self-Assessment

### Q1: Replace CRTP with deducing this

**Answer:**

```cpp

#include <iostream>
#include <string>

// ===== Old CRTP way =====
template<typename Derived>
class ComparableCRTP {
public:
    bool operator!=(const Derived& rhs) const {
        const auto& self = static_cast<const Derived&>(*this);
        return !(self == rhs);
    }
    bool operator>(const Derived& rhs) const {
        const auto& self = static_cast<const Derived&>(*this);
        return rhs < self;
    }
};
struct PointCRTP : ComparableCRTP<PointCRTP> {
    int x, y;
    bool operator==(const PointCRTP& o) const { return x == o.x && y == o.y; }
    bool operator<(const PointCRTP& o) const { return x < o.x || (x == o.x && y < o.y); }
};

// ===== C++23 deducing this =====
class Comparable {
public:
    // 'self' deduces to the actual derived type!
    bool operator!=(this const auto& self, const decltype(self)& rhs) {
        return !(self == rhs);
    }
    bool operator>(this const auto& self, const decltype(self)& rhs) {
        return rhs < self;
    }
};

struct Point : Comparable {
    int x, y;
    bool operator==(const Point& o) const { return x == o.x && y == o.y; }
    bool operator<(const Point& o) const { return x < o.x || (x == o.x && y < o.y); }
};
// No template parameters on Comparable! No static_cast!

```

### Q2: Eliminate const/non-const overload duplication

**Answer:**

```cpp

#include <vector>
#include <string>
#include <iostream>

class Document {
    std::vector<std::string> lines_;
public:
    Document(std::initializer_list<std::string> lines) : lines_(lines) {}

    // OLD: Two overloads needed
    // const std::string& operator[](size_t i) const { return lines_[i]; }
    // std::string& operator[](size_t i) { return lines_[i]; }

    // C++23: ONE function handles both const and non-const
    auto&& operator[](this auto&& self, size_t i) {
        return std::forward_like<decltype(self)>(self.lines_[i]);
    }
    // If 'self' is const Document&, returns const string&
    // If 'self' is Document&, returns string&
    // If 'self' is Document&&, returns string&& (moved!)

    // Also works for begin/end
    auto begin(this auto&& self) {
        return self.lines_.begin();
    }
    auto end(this auto&& self) {
        return self.lines_.end();
    }
};

int main() {
    Document doc{"Hello", "World"};
    doc[0] = "Modified";                // non-const: string&
    const auto& cdoc = doc;
    std::cout << cdoc[0] << "\n";       // const: const string&
    return 0;
}

```

### Q3: Show recursive lambdas and the builder pattern with deducing this

**Answer:**

```cpp

#include <iostream>
#include <vector>
#include <string>

// Recursive lambda: no Y-combinator needed!
void demo_recursive_lambda() {
    // Fibonacci as a lambda
    auto fib = [](this auto&& self, int n) -> int {
        if (n <= 1) return n;
        return self(n - 1) + self(n - 2);  // Recursive!
    };
    std::cout << "fib(10) = " << fib(10) << "\n";  // 55

    // Tree traversal
    struct Node {
        int value;
        std::vector<Node> children;
    };

    Node tree{1, {
        {2, {{4, {}}, {5, {}}}},
        {3, {{6, {}}}}
    }};

    auto visit = [](this auto&& self, const Node& node) -> void {
        std::cout << node.value << " ";
        for (const auto& child : node.children)
            self(child);  // Recursive!
    };
    visit(tree);  // 1 2 4 5 3 6
    std::cout << "\n";
}

// Builder with deducing this: proper return type in hierarchy
class WidgetBuilder {
    std::string name_;
    int width_ = 100, height_ = 100;
public:
    // Deducing this returns the actual derived builder type
    auto&& set_name(this auto&& self, std::string name) {
        self.name_ = std::move(name);
        return std::forward<decltype(self)>(self);
    }
    auto&& set_size(this auto&& self, int w, int h) {
        self.width_ = w;
        self.height_ = h;
        return std::forward<decltype(self)>(self);
    }
    void build() const {
        std::cout << name_ << " " << width_ << "x" << height_ << "\n";
    }
};

class ButtonBuilder : public WidgetBuilder {
    std::string label_;
public:
    auto&& set_label(this auto&& self, std::string label) {
        self.label_ = std::move(label);
        return std::forward<decltype(self)>(self);
    }
};

int main() {
    demo_recursive_lambda();

    // Chaining works through base AND derived methods
    ButtonBuilder()
        .set_name("btn1")       // Returns ButtonBuilder&& (not WidgetBuilder!)
        .set_label("Click me")  // Still ButtonBuilder
        .set_size(200, 50)
        .build();
    return 0;
}

```

---

## Notes

- **Deducing this eliminates CRTP** for most use cases — simpler, no template parameter
- Reduces const/non-const overload pairs to a single function template
- Recursive lambdas become trivial: `[](this auto&& self, auto... args) { self(args...); }`
- `std::forward_like<decltype(self)>()` (C++23) propagates the const/rvalue qualifier correctly
- Compiler support: GCC 14+, Clang 18+, MSVC 19.36+ (all recent)
- Not yet widely adopted in production — but the direction C++ is heading
