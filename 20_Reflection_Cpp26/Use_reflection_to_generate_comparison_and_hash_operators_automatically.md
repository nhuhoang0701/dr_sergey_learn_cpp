# Use reflection to generate comparison and hash operators automatically

**Category:** Reflection (C++26)  
**Item:** #537  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

Reflection can auto-generate `operator==`, `operator<=>`, and `std::hash` by iterating all data members, eliminating boilerplate.

| Generated | Traditional LOC | Reflection LOC | Maintenance |
| --- | --- | --- | --- |
| `operator==` | N comparisons | 1 generic function | Zero |
| `operator<=>` | N comparisons | 1 generic function | Zero |
| `std::hash<T>` | N hash combines | 1 generic specialization | Zero |

---

## Self-Assessment

### Q1: Generic `operator==` that compares all members field-by-field

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>

// Generic equality — works for ANY aggregate:
template <typename T>
bool reflected_equal(const T& a, const T& b) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    bool result = true;
    template for (constexpr auto m : members) {
        if (a.[:m:] != b.[:m:]) result = false;
    }
    return result;
}

struct Point { int x, y, z; };
struct Person { std::string name; int age; double height; };

// Inject operator== via friend:
struct Color {
    int r, g, b, a;
    friend bool operator==(const Color& lhs, const Color& rhs) {
        return reflected_equal(lhs, rhs);
    }
};

int main() {
    Point a{1, 2, 3}, b{1, 2, 3}, c{1, 0, 3};
    std::cout << reflected_equal(a, b) << '\n'; // 1
    std::cout << reflected_equal(a, c) << '\n'; // 0

    Person p1{"Alice", 30, 5.6}, p2{"Alice", 30, 5.6};
    std::cout << reflected_equal(p1, p2) << '\n'; // 1

    Color red{255, 0, 0, 255}, also_red{255, 0, 0, 255};
    std::cout << (red == also_red) << '\n';  // 1 (uses operator==)
}

```

### Q2: Generic `std::hash<T>` for any aggregate via reflection

```cpp

#include <meta>
#include <functional>
#include <iostream>
#include <unordered_set>
#include <string>

// Hash combine utility:
inline std::size_t hash_combine(std::size_t seed, std::size_t h) {
    return seed ^ (h + 0x9e3779b9 + (seed << 6) + (seed >> 2));
}

// Generic hash for any aggregate:
template <typename T>
struct ReflectedHash {
    std::size_t operator()(const T& obj) const {
        constexpr auto members = std::meta::nonstatic_data_members_of(^T);
        std::size_t seed = 0;
        template for (constexpr auto m : members) {
            using MType = [:std::meta::type_of(m):];
            seed = hash_combine(seed, std::hash<MType>{}(obj.[:m:]));
        }
        return seed;
    }
};

struct Point {
    int x, y;
    bool operator==(const Point&) const = default;
};

struct Employee {
    std::string name;
    int id;
    bool operator==(const Employee&) const = default;
};

int main() {
    // Use in unordered containers:
    std::unordered_set<Point, ReflectedHash<Point>> points;
    points.insert({1, 2});
    points.insert({3, 4});
    points.insert({1, 2}); // duplicate, not inserted
    std::cout << "Points: " << points.size() << '\n'; // 2

    std::unordered_set<Employee, ReflectedHash<Employee>> employees;
    employees.insert({"Alice", 1});
    employees.insert({"Bob", 2});
    std::cout << "Employees: " << employees.size() << '\n'; // 2

    // Verify hash consistency:
    ReflectedHash<Point> h;
    std::cout << (h({1,2}) == h({1,2})) << '\n'; // 1 (same input = same hash)
    std::cout << (h({1,2}) == h({2,1})) << '\n'; // 0 (usually different)
}

```

### Q3: Compile-time cost: reflection-generated vs handwritten

```cpp

#include <meta>
#include <iostream>
#include <compare>

struct Vec3 { float x, y, z; };

// ===== Handwritten =====
bool manual_equal(const Vec3& a, const Vec3& b) {
    return a.x == b.x && a.y == b.y && a.z == b.z;
}

auto manual_compare(const Vec3& a, const Vec3& b) {
    if (auto c = a.x <=> b.x; c != 0) return c;
    if (auto c = a.y <=> b.y; c != 0) return c;
    return a.z <=> b.z;
}

// ===== Reflection-generated =====
template <typename T>
bool refl_equal(const T& a, const T& b) {
    bool eq = true;
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
        if (a.[:m:] != b.[:m:]) eq = false;
    }
    return eq;
}

template <typename T>
auto refl_compare(const T& a, const T& b) {
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^T)) {
        if (auto c = a.[:m:] <=> b.[:m:]; c != 0) return c;
    }
    return std::strong_ordering::equal;
}

int main() {
    Vec3 a{1.0f, 2.0f, 3.0f}, b{1.0f, 2.0f, 3.0f};

    // Both produce identical results:
    std::cout << manual_equal(a, b) << '\n';  // 1
    std::cout << refl_equal(a, b) << '\n';    // 1

    // Runtime cost: IDENTICAL.
    // The template for expansion generates the same machine code
    // as the handwritten version. The compiler sees:
    //   refl_equal<Vec3> -> a.x==b.x && a.y==b.y && a.z==b.z
    // which is exactly manual_equal.

    // Compile-time cost: slightly higher for reflection
    // (compiler must resolve meta queries), but usually <1% of
    // total build time. Pays off when you have 50+ types.
}

```

---

## Notes

- `template for` expansion generates code identical to hand-written field-by-field operations.
- Runtime performance of reflection-generated code is the same as manual code after optimization.
- Compile-time cost is slightly higher but negligible for most projects.
- C++20 `operator<=>` with `= default` handles simple cases; reflection handles complex custom logic.
- Hash combine formula (0x9e3779b9) is the golden ratio hash from Boost.
