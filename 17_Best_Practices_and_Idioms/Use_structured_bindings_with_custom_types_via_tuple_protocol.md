# Use structured bindings with custom types via tuple protocol

**Category:** Best Practices & Idioms  
**Item:** #179  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/structured_binding>  

---

## Topic Overview

**Structured bindings** (C++17) unpack objects into named variables. For non-aggregate custom types, you implement the **tuple protocol**: `tuple_size`, `tuple_element`, and `get<N>`.

```cpp

Structured bindings work with:

1. Arrays            → auto [a, b, c] = arr;
2. Aggregates        → auto [x, y] = Point{1, 2};
3. Tuple-like types  → auto [k, v] = std::pair{"a", 1};
4. Custom types      → implement tuple protocol

```

---

## Self-Assessment

### Q1: Implement the tuple protocol for a custom `Point` type

```cpp

#include <iostream>
#include <tuple>
#include <cstddef>

class Point {
    double x_, y_;
public:
    constexpr Point(double x, double y) : x_(x), y_(y) {}
    constexpr double x() const { return x_; }
    constexpr double y() const { return y_; }
};

// Step 1: Specialize std::tuple_size
template<>
struct std::tuple_size<Point> : std::integral_constant<size_t, 2> {};

// Step 2: Specialize std::tuple_element
template<size_t I>
struct std::tuple_element<I, Point> {
    using type = double;
};

// Step 3: Provide get<N> function (in same namespace as Point, or std)
template<size_t I>
constexpr double get(const Point& p) {
    if constexpr (I == 0) return p.x();
    else if constexpr (I == 1) return p.y();
}

int main() {
    Point p(3.0, 4.0);

    // Now structured bindings work!
    auto [x, y] = p;
    std::cout << "x=" << x << " y=" << y << '\n';

    // Works in range-for too
    Point points[] = {{1, 2}, {3, 4}, {5, 6}};
    for (auto [px, py] : points)
        std::cout << "(" << px << ", " << py << ") ";
    std::cout << '\n';

    // Verify sizes at compile time
    static_assert(std::tuple_size_v<Point> == 2);
}
// Expected output:
// x=3 y=4
// (1, 2) (3, 4) (5, 6)

```

### Q2: Verify `auto [x, y] = point;` works with references

```cpp

#include <iostream>
#include <tuple>
#include <cstddef>

class Point {
    double x_, y_;
public:
    Point(double x, double y) : x_(x), y_(y) {}
    double& x() { return x_; }
    double& y() { return y_; }
    const double& x() const { return x_; }
    const double& y() const { return y_; }
};

template<> struct std::tuple_size<Point> : std::integral_constant<size_t, 2> {};
template<size_t I> struct std::tuple_element<I, Point> { using type = double; };

// Mutable get: returns reference
template<size_t I>
double& get(Point& p) {
    if constexpr (I == 0) return p.x();
    else return p.y();
}

// Const get
template<size_t I>
const double& get(const Point& p) {
    if constexpr (I == 0) return p.x();
    else return p.y();
}

int main() {
    Point p(1.0, 2.0);

    // By value (copies)
    auto [x1, y1] = p;
    x1 = 99.0;
    std::cout << "After value copy: p.x=" << p.x() << '\n';  // unchanged

    // By reference (aliases)
    auto& [x2, y2] = p;
    x2 = 10.0;
    y2 = 20.0;
    std::cout << "After ref modify: p=(" << p.x() << ", " << p.y() << ")\n";

    // Const reference
    const auto& [x3, y3] = p;
    std::cout << "Const ref: (" << x3 << ", " << y3 << ")\n";
    // x3 = 5.0;  // ERROR: const!
}
// Expected output:
// After value copy: p.x=1
// After ref modify: p=(10, 20)
// Const ref: (10, 20)

```

### Q3: How standard library types implement structured bindings

```cpp

#include <array>
#include <iostream>
#include <map>
#include <tuple>

int main() {
    // std::pair — has tuple protocol built-in
    std::pair<std::string, int> person{"Alice", 30};
    auto [name, age] = person;
    std::cout << name << " is " << age << '\n';

    // std::tuple — native support
    std::tuple<int, double, char> data{42, 3.14, 'X'};
    auto [id, value, code] = data;
    std::cout << id << " " << value << " " << code << '\n';

    // std::array — works as aggregate
    std::array<int, 3> arr{10, 20, 30};
    auto [a, b, c] = arr;
    std::cout << a << " " << b << " " << c << '\n';

    // std::map iteration — most common use!
    std::map<std::string, int> scores{{"Alice", 95}, {"Bob", 87}};
    for (const auto& [k, v] : scores)
        std::cout << k << ": " << v << '\n';

    // std::map::insert returns pair<iterator, bool>
    auto [it, inserted] = scores.insert({"Carol", 92});
    std::cout << it->first << " inserted=" << inserted << '\n';
}
// Expected output:
// Alice is 30
// 42 3.14 X
// 10 20 30
// Alice: 95
// Bob: 87
// Carol inserted=1

```

---

## Notes

- The tuple protocol requires: `std::tuple_size<T>`, `std::tuple_element<I, T>`, and `get<I>(t)`.
- `get` can be a member function or a free function found via ADL.
- Aggregates (all public members, no bases) work automatically without the tuple protocol.
- Structured bindings are a declaration, not an expression — you can't nest them.
