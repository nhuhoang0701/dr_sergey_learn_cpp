# Know how to correctly implement comparison operators with spaceship operator

**Category:** OOP Design

---

## Topic Overview

C++20's **spaceship operator** (`<=>`) generates all six comparison operators from a single definition:

| Category | Operators | `<=>` return type |
| --- | --- | --- |
| Equality only | `==`, `!=` | `std::strong_equality` (prefer `==` alone) |
| Full ordering | `<`, `>`, `<=`, `>=`, `==`, `!=` | `std::strong_ordering` |
| Partial ordering | Same but allows unordered | `std::partial_ordering` |
| Weak ordering | Same, equivalents not identical | `std::weak_ordering` |

### What the compiler generates

```cpp

auto operator<=>(const T&) const = default;
  ↓ generates ↓
==  !=  <  >  <=  >=   (all six)

Member-wise comparison in declaration order.

```

---

## Self-Assessment

### Q1: Use defaulted and custom spaceship operators

**Answer:**

```cpp

#include <compare>
#include <string>
#include <iostream>
#include <set>

// Simple case: default works perfectly
struct Point {
    int x, y, z;
    auto operator<=>(const Point&) const = default;
};

// Custom: case-insensitive string wrapper
class CIString {
    std::string data_;
public:
    explicit CIString(std::string s) : data_(std::move(s)) {}
    const std::string& str() const { return data_; }

    // Custom spaceship for case-insensitive ordering
    std::weak_ordering operator<=>(const CIString& other) const {
        auto to_lower = [](char c) -> char {
            return (c >= 'A' && c <= 'Z') ? c + 32 : c;
        };
        size_t len = std::min(data_.size(), other.data_.size());
        for (size_t i = 0; i < len; ++i) {
            char a = to_lower(data_[i]);
            char b = to_lower(other.data_[i]);
            if (a != b)
                return a < b ? std::weak_ordering::less
                             : std::weak_ordering::greater;
        }
        return data_.size() <=> other.data_.size();
    }

    // Must define == separately for optimized equality check
    bool operator==(const CIString& other) const {
        if (data_.size() != other.data_.size()) return false;
        return (*this <=> other) == 0;
    }
};

int main() {
    // Point uses default <=> — all operators work
    Point a{1, 2, 3}, b{1, 2, 4};
    std::cout << std::boolalpha;
    std::cout << (a < b) << "\n";   // true
    std::cout << (a == b) << "\n";  // false

    // CIString: case-insensitive ordering
    std::set<CIString> names;
    names.insert(CIString("Alice"));
    names.insert(CIString("alice"));  // Same as "Alice"!
    std::cout << names.size() << "\n";  // 1
    return 0;
}

```

### Q2: Handle mixed-type comparisons and ordering categories

**Answer:**

```cpp

#include <compare>
#include <cmath>
#include <iostream>

// Partial ordering: NaN is unordered
class SafeFloat {
    double val_;
public:
    explicit SafeFloat(double v) : val_(v) {}
    double value() const { return val_; }

    std::partial_ordering operator<=>(const SafeFloat& other) const {
        if (std::isnan(val_) || std::isnan(other.val_))
            return std::partial_ordering::unordered;
        if (val_ < other.val_) return std::partial_ordering::less;
        if (val_ > other.val_) return std::partial_ordering::greater;
        return std::partial_ordering::equivalent;
    }
    bool operator==(const SafeFloat& o) const {
        return !std::isnan(val_) && !std::isnan(o.val_) && val_ == o.val_;
    }
};

// Mixed-type comparison: Meters vs Feet
struct Meters {
    double value;
    std::partial_ordering operator<=>(const Meters& o) const = default;
};
struct Feet { double value; };

// Free function for cross-type comparison
std::partial_ordering operator<=>(const Meters& m, const Feet& f) {
    return m.value <=> (f.value * 0.3048);
}
bool operator==(const Meters& m, const Feet& f) {
    return (m <=> f) == 0;
}

int main() {
    SafeFloat a(1.0), nan(NAN);
    auto r = a <=> nan;
    std::cout << (r == std::partial_ordering::unordered) << "\n";  // true

    Meters m{1.0};
    Feet f{3.28084};
    std::cout << ((m <=> f) == std::partial_ordering::equivalent) << "\n"; // ~true
    return 0;
}

```

### Q3: Implement spaceship for a class with member ordering priority

**Answer:**

```cpp

#include <compare>
#include <string>
#include <iostream>

// Version: compare major, then minor, then patch
struct Version {
    int major, minor, patch;
    std::string label;  // "alpha", "beta", "" for release

    // Custom: ignore label for ordering, but include for equality
    std::strong_ordering operator<=>(const Version& o) const {
        if (auto cmp = major <=> o.major; cmp != 0) return cmp;
        if (auto cmp = minor <=> o.minor; cmp != 0) return cmp;
        return patch <=> o.patch;
    }

    // Equality includes label
    bool operator==(const Version& o) const {
        return major == o.major && minor == o.minor
            && patch == o.patch && label == o.label;
    }
};

int main() {
    Version v1{2, 1, 0, "alpha"};
    Version v2{2, 1, 0, "beta"};
    Version v3{2, 1, 1, ""};

    std::cout << std::boolalpha;
    std::cout << (v1 < v3) << "\n";   // true (2.1.0 < 2.1.1)
    std::cout << (v1 == v2) << "\n";  // false (different labels)
    std::cout << (v1 < v2) << "\n";   // false (same version number)
    std::cout << ((v1 <=> v2) == 0) << "\n";  // true (ordering ignores label)
    return 0;
}

```

---

## Notes

- **Default `<=>` is almost always enough** — compares all members in declaration order
- If you define custom `<=>`, you **must** also define `==` separately for optimization
- `strong_ordering`: equal values are identical (integers, strings)
- `weak_ordering`: equivalent but distinguishable (case-insensitive strings)
- `partial_ordering`: some values are unordered (floating-point NaN)
- Rewrite rule: `a < b` is rewritten as `(a <=> b) < 0` — no need to write individual operators
