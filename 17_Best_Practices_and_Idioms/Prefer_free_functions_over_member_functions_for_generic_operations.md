# Prefer free functions over member functions for generic operations

**Category:** Best Practices & Idioms  
**Item:** #144  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rc-standalone>  

---

## Topic Overview

Free functions enable extension without modification, work with ADL, and can operate generically on types that share an interface but don't share a base class.

### Member vs Free Function Discovery

```cpp

obj.swap(other)    → compiler looks in obj's class only
swap(obj, other)   → compiler looks in:

                     1. Current namespace
                     2. Namespace of obj's type (ADL)
                     3. using declarations

```

---

## Self-Assessment

### Q1: Explain how a non-member `swap` extends the interface without changing the class

```cpp

#include <algorithm>
#include <iostream>
#include <utility>

// Library class (we can't modify it)
class ThirdPartyWidget {
    int id_;
    double data_[100]; // expensive to copy
public:
    ThirdPartyWidget(int id) : id_(id), data_{} {}
    int id() const { return id_; }
    double* data() { return data_; }
    const double* data() const { return data_; }
};

// Extend with efficient swap WITHOUT changing ThirdPartyWidget
void swap(ThirdPartyWidget& a, ThirdPartyWidget& b) noexcept {
    // Use the class's public interface to swap
    ThirdPartyWidget tmp(0);
    std::memcpy(tmp.data(), a.data(), sizeof(double) * 100);
    std::memcpy(a.data(), b.data(), sizeof(double) * 100);
    std::memcpy(b.data(), tmp.data(), sizeof(double) * 100);
    std::cout << "Custom swap called for widgets " << a.id() << " and " << b.id() << '\n';
}

// ADL finds our swap:
template<typename T>
void sort_two(T& a, T& b) {
    using std::swap;  // fallback
    if (b.id() < a.id())
        swap(a, b);   // ADL finds our swap for ThirdPartyWidget
}

int main() {
    ThirdPartyWidget w1(2), w2(1);
    sort_two(w1, w2);
    std::cout << "After sort: " << w1.id() << ", " << w2.id() << '\n';
}
// Expected output:
// Custom swap called for widgets 2 and 1
// After sort: 1, 2

```

### Q2: Show how ADL finds the right free function for a type

```cpp

#include <iostream>
#include <string>

namespace physics {
    struct Vector3 {
        double x, y, z;
    };

    // Free function in the same namespace as Vector3
    double magnitude(const Vector3& v) {
        return std::sqrt(v.x * v.x + v.y * v.y + v.z * v.z);
    }

    std::ostream& operator<<(std::ostream& os, const Vector3& v) {
        return os << "(" << v.x << ", " << v.y << ", " << v.z << ")";
    }
}

namespace graphics {
    struct Color {
        float r, g, b;
    };

    float brightness(const Color& c) {
        return 0.299f * c.r + 0.587f * c.g + 0.114f * c.b;
    }

    std::ostream& operator<<(std::ostream& os, const Color& c) {
        return os << "rgb(" << c.r << ", " << c.g << ", " << c.b << ")";
    }
}

int main() {
    physics::Vector3 v{3, 4, 0};
    graphics::Color c{1.0f, 0.5f, 0.0f};

    // ADL finds the right function based on argument type's namespace
    std::cout << v << " magnitude: " << magnitude(v) << '\n';
    std::cout << c << " brightness: " << brightness(c) << '\n';
    // No 'using' needed! ADL automatically looks in physics:: and graphics::
}
// Expected output:
// (3, 4, 0) magnitude: 5
// rgb(1, 0.5, 0) brightness: 0.5925

```

### Q3: Compare `std::size()`, `std::begin()`, `std::end()` as free functions with `.size()`, `.begin()` as members

```cpp

#include <array>
#include <iostream>
#include <iterator>
#include <vector>

// Free functions work with BOTH containers AND C-arrays:
template<typename Range>
void print_info(const Range& r) {
    std::cout << "Size: " << std::size(r) << ", First: ";
    if (std::size(r) > 0)
        std::cout << *std::begin(r);
    std::cout << '\n';
}

int main() {
    std::vector<int> vec = {1, 2, 3, 4, 5};
    int arr[] = {10, 20, 30};
    std::array<int, 4> sarr = {100, 200, 300, 400};

    // Free functions: work with all three
    print_info(vec);   // std::size(vec)  -> 5
    print_info(arr);   // std::size(arr)  -> 3 (deduced from C-array)
    print_info(sarr);  // std::size(sarr) -> 4

    // Member functions: DON'T work with C-arrays
    // vec.size();  // OK
    // arr.size();  // ERROR: C-array has no .size() method!
}
// Expected output:
// Size: 5, First: 1
// Size: 3, First: 10
// Size: 4, First: 100

```

**Comparison:**

| Function | Member `.size()` | Free `std::size()` |
| --- | --- | --- |
| `vector<T>` | ✓ | ✓ |
| `array<T,N>` | ✓ | ✓ |
| `T arr[N]` (C-array) | ✗ | ✓ |
| `string_view` | ✓ | ✓ |
| `span<T>` | ✓ | ✓ |
| Generic code | Requires `.size()` method | Works with anything |

---

## Notes

- Use `using std::swap;` before unqualified `swap(a, b)` to enable ADL + fallback.
- `std::size()`, `std::begin()`, `std::end()`, `std::data()`, `std::empty()` were added in C++17.
- Ranges (C++20) use `std::ranges::begin()` etc. which handle even more edge cases.
- Free functions + ADL is the C++ way to achieve something like Rust traits or Haskell type classes.
