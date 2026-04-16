# Understand aggregate CTAD and its interaction with designated initializers

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/class_template_argument_deduction>  

---

## Topic Overview

C++20 extends Class Template Argument Deduction to aggregates, allowing template arguments to be deduced from the initializer list — including designated initializers.

### Aggregate CTAD

```cpp

#include <iostream>
#include <string>

template<typename T, typename U>
struct Pair {
    T first;
    U second;
};

int main() {
    // C++17: must specify template arguments
    Pair<int, std::string> p1{42, "hello"};

    // C++20: aggregate CTAD — deduced from initializers
    Pair p2{42, "hello"};             // Pair<int, const char*>
    Pair p3{42, std::string("hi")};   // Pair<int, std::string>

    // Designated initializers + CTAD (C++20):
    Pair p4{.first = 3.14, .second = 42};  // Pair<double, int>

    std::cout << p2.first << " " << p4.first << "\n";
}

```

### With Arrays

```cpp

template<typename T, int N>
struct Array {
    T data[N];
};

// C++20 aggregate CTAD:
Array a1{{1, 2, 3}};  // Array<int, 3> — N deduced from initializer count

// Note: the double braces are needed (outer for aggregate, inner for array)

```

### CTAD Guides for Aggregates

```cpp

#include <cstddef>
#include <type_traits>

template<typename T, size_t N>
struct FixedString {
    T data[N];
};

// Deduction guide to deduce N from string literal
template<typename T, size_t N>
FixedString(const T (&)[N]) -> FixedString<T, N>;

auto fs = FixedString{"hello"};  // FixedString<char, 6> (including null)

```

---

## Self-Assessment

### Q1: What aggregates qualify for CTAD

A class template is an aggregate if it has no user-declared constructors, no private/protected non-static data members, no virtual base classes, and no virtual functions. All public members participate in deduction.

### Q2: Show where aggregate CTAD fails

```cpp

template<typename T>
struct Wrapper {
    T value;
    int count = 0;  // Default member initializer
};

Wrapper w{42};  // OK: T = int, count = 0
Wrapper w2{};   // Error: can't deduce T from nothing

```

### Q3: How do designated initializers interact with CTAD

Designated initializers participate in deduction for the fields they name. The order must match declaration order (C++ requirement), and skipping fields uses their default values. CTAD uses the designated fields to deduce template parameters.

---

## Notes

- Aggregate CTAD works with both brace-init and designated initializers.
- Requires C++20 — not available in C++17 even with deduction guides.
- Combined with `std::to_array`, enables concise aggregate creation.
- Some compilers had delayed support — check GCC 11+, Clang 17+, MSVC 19.30+.
