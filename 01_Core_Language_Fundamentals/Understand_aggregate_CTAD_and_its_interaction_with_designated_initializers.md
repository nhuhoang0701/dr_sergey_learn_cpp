# Understand aggregate CTAD and its interaction with designated initializers

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/class_template_argument_deduction>  

---

## Topic Overview

C++20 extends Class Template Argument Deduction (CTAD) to aggregates, letting template
arguments be deduced from the initializer list - including from designated initializers.
Before C++20 you had to spell out every template argument on an aggregate template; now the
compiler can figure them out from what you give it.

### Aggregate CTAD

Here's the difference between C++17 and C++20 for the same aggregate template. The C++20
forms are much less noisy:

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

    // C++20: aggregate CTAD - deduced from initializers
    Pair p2{42, "hello"};             // Pair<int, const char*>
    Pair p3{42, std::string("hi")};   // Pair<int, std::string>

    // Designated initializers + CTAD (C++20):
    Pair p4{.first = 3.14, .second = 42};  // Pair<double, int>

    std::cout << p2.first << " " << p4.first << "\n";
}
```

Notice that `p4` uses designated initializers and CTAD at the same time - the compiler
deduces `T = double` and `U = int` directly from the named fields.

### With Arrays

For array-member aggregates, CTAD can also deduce the non-type template parameter for
the array size. The double braces are required: the outer pair is for the aggregate, the
inner pair is for the array member:

```cpp
template<typename T, int N>
struct Array {
    T data[N];
};

// C++20 aggregate CTAD:
Array a1{{1, 2, 3}};  // Array<int, 3> - N deduced from initializer count

// Note: the double braces are needed (outer for aggregate, inner for array)
```

### CTAD Guides for Aggregates

When the automatic deduction isn't quite right, you can add an explicit deduction guide.
This example teaches the compiler how to deduce `N` from a string literal's length:

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

A class template is an aggregate if it has no user-declared constructors, no private or
protected non-static data members, no virtual base classes, and no virtual functions. When
those conditions are met, all public members participate in deduction - the compiler reads
the initializer list and matches each value to the corresponding member's type.

### Q2: Show where aggregate CTAD fails

Deduction needs something to work from. If you provide no initializer at all for the field
whose type you want deduced, the compiler has nothing to go on:

```cpp
template<typename T>
struct Wrapper {
    T value;
    int count = 0;  // Default member initializer
};

Wrapper w{42};  // OK: T = int, count = 0
Wrapper w2{};   // Error: can't deduce T from nothing
```

The second line fails because there's no initializer for `value`, so `T` is ambiguous.

### Q3: How do designated initializers interact with CTAD

Designated initializers participate in deduction for the fields they name. The order must
match declaration order (C++ requirement), and skipping fields uses their default values.
CTAD uses the designated fields to deduce template parameters - so naming `.first = 3.14`
is enough for the compiler to conclude `T = double`, even without an explicit template
argument list.

---

## Notes

- Aggregate CTAD works with both brace-init and designated initializers.
- Requires C++20 - not available in C++17 even with deduction guides.
- Combined with `std::to_array`, enables concise aggregate creation.
- Some compilers had delayed support - check GCC 11+, Clang 17+, MSVC 19.30+.
