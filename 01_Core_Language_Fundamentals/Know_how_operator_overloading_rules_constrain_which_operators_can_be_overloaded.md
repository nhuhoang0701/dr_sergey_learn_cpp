# Know how operator overloading rules constrain which operators can be overloaded

**Category:** Core Language Fundamentals  
**Reference:** <https://en.cppreference.com/w/cpp/language/operators>  

---

## Topic Overview

### What Is Operator Overloading

C++ lets you redefine the behavior of operators (`+`, `-`, `<<`, `[]`, etc.) for user-defined types. However, there are strict rules governing **which** operators can be overloaded, **how** they must be declared, and **what constraints** apply.

### Operators That CANNOT Be Overloaded

| Operator | Name | Reason |
| --- | --- | --- |
| `::` | Scope resolution | Fundamental to name lookup, not a runtime operation |
| `.` | Member access | Must always mean "access member" — no alternative semantics |
| `.*` | Pointer-to-member access | Same reason as `.` |
| `?:` | Ternary conditional | Has unique short-circuit semantics that can't be preserved |
| `sizeof` | Size query | Compile-time operator, not a function call |
| `typeid` | Type identification | Compile-time/RTTI operator |
| `alignof` | Alignment query | Compile-time operator |
| `noexcept` | Noexcept check | Compile-time operator |

### Member vs. Free Function Rules

| Operator | Must be member? | Typically free? | Notes |
| --- | --- | --- | --- |
| `=` (assignment) | **Yes** | — | Prevents hijacking assignment semantics |
| `()` (call) | **Yes** | — | Defines function objects/functors |
| `[]` (subscript) | **Yes** | — | C++23 allows multiple parameters |
| `->` (member access) | **Yes** | — | Must return pointer or proxy |
| `->*` | No | No | Rarely overloaded |
| `<<`, `>>` (stream) | No | **Yes** | Left operand is `std::ostream&`, not your type |
| `+`, `-`, `*`, `/` | No | **Yes** | Free function enables symmetric conversions |
| `==`, `<=>` | **Yes** (C++20) | — | Compiler generates `!=`, `<`, `>`, etc. |
| Conversion `operator T()` | **Yes** | — | Implicit or explicit conversion |
| `new`, `delete` | May be member or global | — | Controls allocation |

### Why `operator=` Must Be a Member

If `operator=` could be a free function, external code could hijack the assignment semantics of a class:

```cpp

// If this were allowed (it's NOT):
MyClass& operator=(MyClass& lhs, const MyClass& rhs);  // ERROR: = must be member

```

The compiler would have no way to guarantee the class maintains its invariants during assignment.

### Why `operator<<` Is Typically Free

```cpp

struct Point {
    double x, y;

    // Can't make operator<< a member — the left operand is std::ostream, not Point!
    // std::ostream& operator<<(std::ostream& os) would mean: point << std::cout; // wrong order!
};

// Correct: free function (or hidden friend)
std::ostream& operator<<(std::ostream& os, const Point& p) {
    return os << "(" << p.x << ", " << p.y << ")";
}
// Usage: std::cout << point;  // natural syntax

```

### The Canonical Operator Patterns

**Arithmetic operators:** Implement `+=` as member, `+` as free function:

```cpp

struct Vec2 {
    double x, y;

    // Member: modifies this object
    Vec2& operator+=(const Vec2& rhs) {
        x += rhs.x;
        y += rhs.y;
        return *this;
    }

    // C++20 spaceship operator for all comparisons
    auto operator<=>(const Vec2&) const = default;
};

// Free function: creates a new object (enables implicit conversion on both sides)
Vec2 operator+(Vec2 lhs, const Vec2& rhs) {
    lhs += rhs;  // Reuses operator+= (lhs is a copy!)
    return lhs;
}

```

**Unary operators:**

```cpp

struct Integer {
    int val;

    Integer  operator-() const { return Integer{-val}; }       // Unary minus
    Integer  operator+() const { return *this; }                // Unary plus
    Integer& operator++()      { ++val; return *this; }        // Pre-increment
    Integer  operator++(int)   { auto old = *this; ++val; return old; } // Post-increment
    bool     operator!() const { return val == 0; }             // Logical NOT
};

```

### C++20 Comparison Operators

C++20 dramatically simplifies comparison operators:

```cpp

#include <compare>

struct Point3D {
    double x, y, z;

    // Generates ==, !=, <, >, <=, >= automatically!
    auto operator<=>(const Point3D&) const = default;
};

```

If you need custom logic:

```cpp

struct CaseInsensitiveString {
    std::string data;

    bool operator==(const CaseInsensitiveString& other) const {
        return std::ranges::equal(data, other.data, [](char a, char b) {
            return std::tolower(a) == std::tolower(b);
        });
    }

    std::weak_ordering operator<=>(const CaseInsensitiveString& other) const {
        // Custom comparison logic
        auto a = data, b = other.data;
        std::transform(a.begin(), a.end(), a.begin(), ::tolower);
        std::transform(b.begin(), b.end(), b.begin(), ::tolower);
        return a <=> b;
    }
};

```

### Overloading Pitfalls

1. **Don't overload `&&`, `||`, or `,`** — they lose short-circuit evaluation and sequencing guarantees.
2. **Don't change operator semantics** — `operator+` should add, not subtract.
3. **Be consistent** — if you overload `==`, overload `!=` too (or use `= default` in C++20).
4. **Return types matter** — `operator+` should return by value, `operator+=` by reference.

---

## Self-Assessment

### Q1: List operators that cannot be overloaded

The following operators **cannot** be overloaded in C++:

| Operator | Symbol | Why |
| --- | --- | --- |
| Scope resolution | `::` | Compile-time name lookup, not a runtime operation |
| Member access | `.` | Must always access the actual member, no alternative meaning |
| Pointer-to-member | `.*` | Same constraint as `.` |
| Ternary conditional | `?:` | Unique short-circuit semantics can't be replicated in a function |
| `sizeof` | `sizeof` | Compile-time query, not a function |
| `typeid` | `typeid` | RTTI operator, not overloadable |
| `alignof` | `alignof` | Compile-time alignment query |
| `noexcept` | `noexcept` | Compile-time exception specification check |

Additionally, these preprocessing operators cannot be overloaded: `#` (stringize), `##` (token paste).

### Q2: Explain why `operator=` must be a member and `operator<<` is typically free

**`operator=` must be a non-static member function because:**

1. **Invariant protection:** The class author must control assignment to maintain class invariants (e.g., resource ownership, reference counts).
2. **Default generation:** The compiler generates `operator=` if you don't — this only works for member functions.
3. **Preventing hijacking:** If external code could define `operator=` for your class, it could silently break your type's semantics.

```cpp

struct Resource {
    int* ptr;
    // Only the class author should decide how assignment works
    Resource& operator=(const Resource& other) {
        if (this != &other) {
            delete ptr;
            ptr = new int(*other.ptr);
        }
        return *this;
    }
};

```

**`operator<<` is typically a free function (or hidden friend) because:**

1. **Left operand isn't your type:** `std::cout << obj` means `operator<<(std::cout, obj)`. You can't add a member to `std::ostream`.
2. **If it were a member of your class,** the syntax would be backwards: `obj << std::cout` (left operand = `this`).
3. **Hidden friend pattern** is preferred in modern C++:

```cpp

struct Color {
    int r, g, b;
    friend std::ostream& operator<<(std::ostream& os, const Color& c) {
        return os << "rgb(" << c.r << "," << c.g << "," << c.b << ")";
    }
};

```

### Q3: Show the canonical form for overloading `operator+` as a free function using `operator+=`

```cpp

#include <iostream>
#include <cmath>

struct Complex {
    double real, imag;

    // Constructor
    Complex(double r = 0, double i = 0) : real(r), imag(i) {}

    // operator+= as MEMBER (modifies *this)
    Complex& operator+=(const Complex& rhs) {
        real += rhs.real;
        imag += rhs.imag;
        return *this;
    }

    Complex& operator-=(const Complex& rhs) {
        real -= rhs.real;
        imag -= rhs.imag;
        return *this;
    }

    Complex& operator*=(const Complex& rhs) {
        double new_real = real * rhs.real - imag * rhs.imag;
        double new_imag = real * rhs.imag + imag * rhs.real;
        real = new_real;
        imag = new_imag;
        return *this;
    }

    // Unary minus
    Complex operator-() const { return {-real, -imag}; }

    // Comparison (C++20)
    bool operator==(const Complex&) const = default;

    // Stream output as hidden friend
    friend std::ostream& operator<<(std::ostream& os, const Complex& c) {
        return os << c.real << (c.imag >= 0 ? "+" : "") << c.imag << "i";
    }
};

// operator+ as FREE FUNCTION — takes lhs BY VALUE (it's a copy!)
Complex operator+(Complex lhs, const Complex& rhs) {
    lhs += rhs;     // Reuse operator+=
    return lhs;     // Return the modified copy
}

Complex operator-(Complex lhs, const Complex& rhs) {
    lhs -= rhs;
    return lhs;
}

Complex operator*(Complex lhs, const Complex& rhs) {
    lhs *= rhs;
    return lhs;
}

int main() {
    Complex a{3, 4}, b{1, -2};

    std::cout << "a     = " << a << "\n";       // 3+4i
    std::cout << "b     = " << b << "\n";       // 1-2i
    std::cout << "a + b = " << (a + b) << "\n"; // 4+2i
    std::cout << "a - b = " << (a - b) << "\n"; // 2+6i
    std::cout << "a * b = " << (a * b) << "\n"; // 11-2i
    std::cout << "-a    = " << (-a) << "\n";    // -3-4i

    // Chaining works:
    Complex c = a + b + Complex{10, 0};
    std::cout << "a+b+10 = " << c << "\n";      // 14+2i

    return 0;
}

```

**Why this pattern is canonical:**

1. **`operator+=` as member** — modifies `*this`, returns `*this` by reference. Natural semantics.
2. **`operator+` as free function** — takes `lhs` **by value** (making a copy), applies `+=`, returns the copy.
3. **No code duplication** — the addition logic is only in `operator+=`.
4. **Symmetric conversions** — as a free function, implicit conversions work for both operands.
5. **Efficient** — the copy can be elided (NRVO) in many cases.

---

## Additional Examples

### Subscript Operator (C++23 Multidimensional)

```cpp

#include <vector>
#include <stdexcept>
#include <iostream>

class Matrix {
    std::vector<double> data_;
    std::size_t rows_, cols_;
public:
    Matrix(std::size_t r, std::size_t c) : data_(r * c, 0.0), rows_(r), cols_(c) {}

    // C++23: multidimensional subscript
    double& operator[](std::size_t r, std::size_t c) {
        return data_[r * cols_ + c];
    }
    const double& operator[](std::size_t r, std::size_t c) const {
        return data_[r * cols_ + c];
    }

    // Pre-C++23 fallback: operator()(r, c)
    double& operator()(std::size_t r, std::size_t c) {
        return data_[r * cols_ + c];
    }
};

int main() {
    Matrix m(3, 3);
    m[1, 2] = 42.0;             // C++23 multidimensional subscript
    std::cout << m[1, 2] << "\n"; // 42
}

```

### Function Call Operator (Functor)

```cpp

#include <iostream>
#include <algorithm>
#include <vector>

struct Threshold {
    double limit;
    bool operator()(double val) const {  // Makes Threshold a functor
        return val > limit;
    }
};

int main() {
    std::vector<double> data = {1.5, 3.7, 2.1, 4.8, 0.3};
    auto count = std::count_if(data.begin(), data.end(), Threshold{3.0});
    std::cout << count << " values exceed 3.0\n";  // 2
}

```

---

## Notes

- Use the **canonical patterns**: `+=` as member, `+` as free function reusing `+=`.
- In C++20, prefer `operator<=>` with `= default` for comparisons.
- Use **hidden friends** for stream operators and ADL-only operators.
- Never overload `&&`, `||`, or `,` — they lose special evaluation semantics.
- `operator[]` gained multi-argument support in C++23.
