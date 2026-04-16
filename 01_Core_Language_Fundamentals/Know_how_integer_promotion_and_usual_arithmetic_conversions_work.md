# Know how integer promotion and usual arithmetic conversions work

**Category:** Core Language Fundamentals  
**Standard:** C++98/C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/implicit_conversion>  

---

## Topic Overview

### What Are Integer Promotions

**Integer promotion** is an implicit conversion the compiler performs on "small" integer types before almost any arithmetic or bitwise operation. The rules (inherited from C) are:

| Source type | Promoted to |
| --- | --- |
| `bool`, `char`, `signed char`, `unsigned char`, `short`, `unsigned short` | `int` (if `int` can represent all values), else `unsigned int` |
| Unscoped enum with underlying type smaller than `int` | `int` or `unsigned int` |
| Bit-field with width < `int` width | `int` or `unsigned int` |

**Key insight:** No arithmetic is ever performed on types smaller than `int`. Even `char + char` yields `int`.

```cpp

char a = 100, b = 100;
auto result = a + b;          // result is int (200), NOT char
static_assert(std::is_same_v<decltype(a + b), int>);

```

### Usual Arithmetic Conversions

When a binary operator has operands of **different types**, both are converted to a **common type** by these rules (applied in order):

1. If either operand is `long double` → the other becomes `long double`.
2. Else if either is `double` → the other becomes `double`.
3. Else if either is `float` → the other becomes `float`.
4. Else **integer promotions** are applied to both, then:
   - If both have the same signedness → convert to the wider type.
   - If the unsigned type is ≥ rank of the signed type → convert signed to unsigned. **(This is the dangerous case!)**
   - If the signed type can represent all values of the unsigned type → convert unsigned to signed.
   - Otherwise → both become the unsigned version of the signed type.

### The Signed/Unsigned Comparison Trap

```cpp

int    s = -1;
unsigned u = 2;
// Step 1: Both are already >= int rank, no integer promotion needed
// Step 2: Different signedness, unsigned int rank == signed int rank
// Step 3: Convert signed to unsigned → -1 becomes UINT_MAX (4294967295)
// Step 4: 4294967295 > 2 → true!
if (s > u) {
    // This EXECUTES! -1 > 2u is true!
}

```

This is one of the most common sources of bugs in C/C++ programs.

### Compiler Warnings

GCC/Clang with `-Wsign-compare` (included in `-Wall`) will warn:

```text

warning: comparison of integer expressions of different signedness

```

MSVC with `/W4` also warns about signed/unsigned mismatches.

---

## Self-Assessment

### Q1: Show a signed/unsigned comparison bug where `-1 > 2u` evaluates to `true`

```cpp

#include <iostream>

int main() {
    int signed_val = -1;
    unsigned int unsigned_val = 2;

    // The compiler converts signed_val to unsigned before comparison.
    // -1 as unsigned int = 4294967295 (UINT_MAX on 32-bit int)
    // 4294967295 > 2 → true
    if (signed_val > unsigned_val) {
        std::cout << "-1 > 2u is TRUE (surprising!)\n";
    } else {
        std::cout << "-1 > 2u is false\n";
    }

    // Demonstrating the actual converted value:
    std::cout << "signed_val as unsigned: "
              << static_cast<unsigned int>(signed_val) << "\n";
    // Output: 4294967295

    // Same problem with different types:
    short s = -1;
    unsigned int u = 0;
    std::cout << "(short)-1 > 0u: " << (s > u) << "\n";  // true!
    // short -1 → int -1 (promotion) → unsigned int UINT_MAX (conversion)

    return 0;
}

```

**Output:**

```text

-1 > 2u is TRUE (surprising!)
signed_val as unsigned: 4294967295
(short)-1 > 0u: 1

```

**How this works:**

- When comparing `int` to `unsigned int`, the `int` is converted to `unsigned int`.
- `-1` in two's complement becomes `UINT_MAX` (all bits set = 4294967295).
- `4294967295 > 2` → the comparison evaluates to `true`.
- This affects all relational and equality operators: `<`, `>`, `<=`, `>=`, `==`, `!=`.
- The same problem occurs with `short` vs `unsigned int` — the `short` is first promoted to `int`, then converted to `unsigned int`.

### Q2: Explain what happens when char arithmetic overflows and how to avoid UB

**Whether `char` is signed or unsigned is implementation-defined.** On most platforms (x86), `char` is signed (`-128` to `127`).

**Scenario: Signed `char` overflow:**

```cpp

signed char a = 120;
signed char b = 20;
// a + b performs INTEGER PROMOTION first:
// int(120) + int(20) = int(140) → result is int, no overflow!
signed char c = a + b;  // 140 doesn't fit in signed char → implementation-defined narrowing

```

**Wait — is there UB?** Not in the arithmetic itself! Integer promotion means `a + b` is computed as `int`. The potential issue is when you **store the result back** into a `char`:

- The narrowing conversion from `int` to `signed char` is **implementation-defined** (not UB).
- However, incrementing a `signed char` directly can cause issues in loops:

```cpp

// UB: signed integer overflow (though promoted to int, the logic error remains)
for (signed char i = 0; i < 200; ++i) {  // i can never reach 200!
    // Infinite loop: i goes 0..127, then wraps (impl-defined) to -128, etc.
}

```

**For `unsigned char`:** arithmetic wraps modulo 256, well-defined.

**How to avoid problems:**

1. **Use `int` or larger types** for arithmetic — don't do math on `char`.
2. **Use `unsigned char` or `std::uint8_t`** when you need byte manipulation.
3. **Enable compiler warnings** (`-Wconversion`, `-Wsign-conversion`).
4. **Use `static_cast` explicitly** when narrowing is intentional.

```cpp

#include <cstdint>
#include <iostream>

int main() {
    // GOOD: Use int for arithmetic
    int sum = 0;
    for (int i = 0; i < 1000; ++i) sum += i;

    // GOOD: Use uint8_t for byte data
    std::uint8_t byte = 200;
    std::uint8_t result = static_cast<std::uint8_t>(byte + 100);  // Wraps to 44, well-defined

    // GOOD: Explicit cast when narrowing
    int big = 300;
    auto small = static_cast<signed char>(big);  // Intentional truncation
    std::cout << static_cast<int>(small) << "\n"; // Implementation-defined value
    return 0;
}

```

### Q3: Demonstrate how to use `std::cmp_less` (C++20) to safely compare signed and unsigned integers

```cpp

#include <iostream>
#include <utility>  // std::cmp_less, std::cmp_greater, etc.

int main() {
    int    signed_val   = -1;
    unsigned unsigned_val = 2;

    // DANGEROUS: raw comparison (signed converted to unsigned)
    std::cout << "Raw:      -1 > 2u  = " << (signed_val > unsigned_val) << "\n";  // 1 (true!)

    // SAFE: std::cmp_less and friends (C++20)
    std::cout << "cmp_less: -1 < 2u  = " << std::cmp_less(signed_val, unsigned_val) << "\n";     // 1 (true)
    std::cout << "cmp_greater: -1 > 2u = " << std::cmp_greater(signed_val, unsigned_val) << "\n"; // 0 (false)
    std::cout << "cmp_equal: -1 == 2u = " << std::cmp_equal(signed_val, unsigned_val) << "\n";    // 0 (false)

    // Practical use: safe bounds checking
    int index = -5;
    std::size_t container_size = 10;

    if (std::cmp_less(index, container_size) && std::cmp_greater_equal(index, 0)) {
        std::cout << "Index is in bounds\n";
    } else {
        std::cout << "Index is out of bounds\n";
    }

    // All C++20 safe comparison functions:
    // std::cmp_equal(a, b)         — true if mathematically a == b
    // std::cmp_not_equal(a, b)     — true if mathematically a != b
    // std::cmp_less(a, b)          — true if mathematically a < b
    // std::cmp_greater(a, b)       — true if mathematically a > b
    // std::cmp_less_equal(a, b)    — true if mathematically a <= b
    // std::cmp_greater_equal(a, b) — true if mathematically a >= b
    // std::in_range<T>(value)      — true if value fits in T

    // std::in_range example:
    std::cout << "Can -1 fit in unsigned? " << std::in_range<unsigned>(-1) << "\n";  // 0 (false)
    std::cout << "Can 42 fit in unsigned? " << std::in_range<unsigned>(42) << "\n";  // 1 (true)
    std::cout << "Can 300 fit in uint8_t? " << std::in_range<uint8_t>(300) << "\n";  // 0 (false)

    return 0;
}

```

**Output:**

```text

Raw:      -1 > 2u  = 1
cmp_less: -1 < 2u  = 1
cmp_greater: -1 > 2u = 0
cmp_equal: -1 == 2u = 0
Index is out of bounds
Can -1 fit in unsigned? 0
Can 42 fit in unsigned? 1
Can 300 fit in uint8_t? 0

```

**How this works:**

- `std::cmp_less` and friends perform **mathematically correct** comparisons regardless of signedness.
- They work by checking the sign first: if the signed value is negative and the unsigned is non-negative, the signed value is always less.
- `std::in_range<T>(value)` checks if a value can be represented by type `T` — perfect for validating before narrowing conversions.
- These are **zero-cost abstractions** — they compile to the same code a careful programmer would write by hand.

---

## Additional Examples

### Integer Promotion Surprises

```cpp

#include <iostream>
#include <type_traits>

int main() {
    unsigned short a = 65535;
    unsigned short b = 1;

    // a + b promotes both to int (assuming int is 32-bit)
    auto sum = a + b;
    static_assert(std::is_same_v<decltype(sum), int>);  // NOT unsigned short!
    std::cout << "65535us + 1us = " << sum << " (type: int)\n";  // 65536

    // But on a platform where int is 16-bit (rare):
    // both would promote to unsigned int, and 65535 + 1 = 0 (wraps)

    // Bitwise NOT on unsigned char:
    unsigned char mask = 0x0F;
    auto notmask = ~mask;  // promotes to int first, then NOT
    // notmask = ~(int)0x0F = 0xFFFFFFF0 (int, negative!)
    static_assert(std::is_same_v<decltype(notmask), int>);
    std::cout << "~0x0F as int: " << notmask << "\n";  // -16

    // To get unsigned char result:
    unsigned char correct = static_cast<unsigned char>(~mask);  // 0xF0
    std::cout << "~0x0F as uchar: " << static_cast<int>(correct) << "\n";  // 240
}

```

### Rank-Based Conversion Table

```cpp

#include <iostream>
#include <typeinfo>
#include <cstdint>

// Helper to print the type of an expression
#define SHOW_TYPE(expr) \
    std::cout << #expr << " has type: " << typeid(expr).name() << " = " << (expr) << "\n"

int main() {
    SHOW_TYPE(1 + 1.0);       // double (int → double)
    SHOW_TYPE(1.0f + 1.0);    // double (float → double)
    SHOW_TYPE(1u + (-2));      // unsigned int (int → unsigned)
    SHOW_TYPE('A' + 1);        // int (char promoted to int)
    SHOW_TYPE(true + true);    // int (bool promoted to int: 1 + 1 = 2)
}

```

---

## Notes

- Always enable `-Wsign-compare` and `-Wconversion` to catch promotion issues.
- Use `std::cmp_*` functions (C++20) for any signed/unsigned comparison in your code.
- Prefer `int` or `std::size_t` for loop counters — avoid arithmetic on `char` or `short`.
- The `auto` keyword preserves the promoted type, which can surprise: `unsigned char a = 0xFF; auto b = ~a;` gives `int`, not `unsigned char`.
- Use `std::in_range<T>()` before narrowing conversions to validate at runtime.
