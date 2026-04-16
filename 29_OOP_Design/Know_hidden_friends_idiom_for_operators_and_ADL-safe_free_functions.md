# Know hidden friends idiom for operators and ADL-safe free functions

**Category:** OOP Design

---

## Topic Overview

A **hidden friend** is a `friend` function defined *inside* a class body. It's only found via **ADL** (Argument-Dependent Lookup), not by normal unqualified lookup:

```cpp

class MyType {
    friend bool operator==(const MyType& a, const MyType& b) { // Hidden friend
        return a.val_ == b.val_;
    }
    int val_;
};

```

| Property | Member operator | Free function | Hidden friend |
| --- | :---: | :---: | :---: |
| Symmetric operands | No (`a==b` ≠ `b==a` for conversions) | **Yes** | **Yes** |
| Found by ADL only | N/A | No (found everywhere) | **Yes** |
| Reduces overload set | N/A | No | **Yes** (faster compile) |
| Access to private | Yes | Needs friend decl | **Yes** |

---

## Self-Assessment

### Q1: Implement operators as hidden friends

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <compare>

class Temperature {
    double kelvin_;
public:
    explicit Temperature(double k) : kelvin_(k) {}
    double kelvin() const { return kelvin_; }

    // Hidden friends: found ONLY via ADL on Temperature
    friend auto operator<=>(const Temperature& a, const Temperature& b) = default;

    friend Temperature operator+(const Temperature& a, const Temperature& b) {
        return Temperature(a.kelvin_ + b.kelvin_);
    }

    friend Temperature operator-(const Temperature& a, const Temperature& b) {
        return Temperature(a.kelvin_ - b.kelvin_);
    }

    friend std::ostream& operator<<(std::ostream& os, const Temperature& t) {
        return os << t.kelvin_ << "K";
    }

    // Hidden friend swap — found by ADL, no std:: needed
    friend void swap(Temperature& a, Temperature& b) noexcept {
        using std::swap;
        swap(a.kelvin_, b.kelvin_);
    }
};

int main() {
    Temperature t1(273.15), t2(373.15);
    std::cout << (t1 < t2) << "\n";    // true
    std::cout << (t1 + t2) << "\n";    // 646.3K
    std::cout << t1 << "\n";           // 273.15K
    return 0;
}

```

### Q2: Show why hidden friends reduce compilation time and prevent surprises

**Answer:**

```cpp

#include <string>
#include <iostream>

namespace lib {
    class Widget {
        int id_;
    public:
        explicit Widget(int id) : id_(id) {}

        // BAD: Free function operator in namespace
        // Found during ANY lookup if namespace is pulled in
    };
    // Non-hidden: pollutes overload resolution
    // bool operator==(const Widget& a, const Widget& b) { return a.id_ == b.id_; }

    class BetterWidget {
        int id_;
    public:
        explicit BetterWidget(int id) : id_(id) {}

        // GOOD: Hidden friend — only found via ADL
        friend bool operator==(const BetterWidget& a, const BetterWidget& b) {
            return a.id_ == b.id_;
        }
    };
}

// The hidden friend for BetterWidget won't accidentally participate
// in overload resolution for unrelated types
struct Unrelated {};

int main() {
    lib::BetterWidget a(1), b(2);
    std::cout << (a == b) << "\n";  // OK: ADL finds operator==

    // Unrelated u1{}, u2{};
    // u1 == u2;  // BetterWidget's operator== is NOT even considered
    //            // (unlike a namespace-scope operator)
    return 0;
}

```

### Q3: Implement a full type with hidden friends for all common operations

**Answer:**

```cpp

#include <iostream>
#include <functional>
#include <string>

class Identifier {
    std::string prefix_;
    int id_;
public:
    Identifier(std::string prefix, int id)
        : prefix_(std::move(prefix)), id_(id) {}

    // All operators as hidden friends
    friend bool operator==(const Identifier& a, const Identifier& b) {
        return a.prefix_ == b.prefix_ && a.id_ == b.id_;
    }

    friend auto operator<=>(const Identifier& a, const Identifier& b) {
        if (auto cmp = a.prefix_ <=> b.prefix_; cmp != 0)
            return cmp;
        return a.id_ <=> b.id_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Identifier& id) {
        return os << id.prefix_ << "-" << id.id_;
    }

    friend std::istream& operator>>(std::istream& is, Identifier& id) {
        char dash;
        return is >> id.prefix_ >> dash >> id.id_;
    }

    // Hidden friend hash support
    friend struct std::hash<Identifier>;

    friend void swap(Identifier& a, Identifier& b) noexcept {
        using std::swap;
        swap(a.prefix_, b.prefix_);
        swap(a.id_, b.id_);
    }
};

// Hash specialization using friend access
template<> struct std::hash<Identifier> {
    size_t operator()(const Identifier& id) const {
        return std::hash<std::string>{}(id.prefix_)
             ^ (std::hash<int>{}(id.id_) << 1);
    }
};

#include <set>
#include <unordered_set>

int main() {
    std::set<Identifier> ordered;
    ordered.emplace("USR", 1);
    ordered.emplace("USR", 2);
    ordered.emplace("ORD", 1);
    for (const auto& id : ordered) std::cout << id << " ";
    std::cout << "\n";  // ORD-1 USR-1 USR-2

    std::unordered_set<Identifier> hashed;
    hashed.emplace("USR", 1);
    return 0;
}

```

---

## Notes

- **Default to hidden friends for all operators** — it's the modern C++ best practice (endorsed by P1601)
- Hidden friends reduce overload set size → faster compilation in large codebases
- They provide **symmetric** operator behavior (both operands equal for implicit conversions)
- `swap`, `operator<<`, `operator>>`, and comparison operators are prime candidates
- The technique is especially important in library code where namespace pollution affects clients
- `friend` defined inside the class = hidden friend; `friend` declared inside + defined outside = NOT hidden
