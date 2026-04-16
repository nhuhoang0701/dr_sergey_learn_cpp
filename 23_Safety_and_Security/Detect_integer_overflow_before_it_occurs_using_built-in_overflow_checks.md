# Detect integer overflow before it occurs using built-in overflow checks

**Category:** Safety & Security  
**Item:** #557  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Integer-Overflow-Builtins.html>  

---

## Topic Overview

This topic focuses on building safe arithmetic wrappers using overflow builtins — complementing #733 (compile-time detection, UBSan, compiler UB exploitation).

```cpp

__builtin_add_overflow(a, b, &result):

  1. Computes a + b
  2. Stores result in *result (even if overflow)
  3. Returns true if overflow occurred
  4. NO undefined behavior!

  Compiled to:
    add eax, ebx   ; compute sum
    jo  overflow    ; jump if overflow flag set
    -> Cost: 1 extra instruction (branch on OF flag)

```

| Builtin | Operation | Available |
| --- | --- | --- |
| `__builtin_add_overflow(a, b, &r)` | Addition | GCC 5+, Clang 3.8+ |
| `__builtin_sub_overflow(a, b, &r)` | Subtraction | GCC 5+, Clang 3.8+ |
| `__builtin_mul_overflow(a, b, &r)` | Multiplication | GCC 5+, Clang 3.8+ |
| `_addcarry_u64` (MSVC) | Addition + carry | MSVC (intrinsic) |
| `std::add_overflow` (C++26) | Addition | C++26 standard |

---

## Self-Assessment

### Q1: __builtin_add_overflow usage

```cpp

#include <iostream>
#include <climits>
#include <cstdint>

int main() {
    // Basic usage: detect addition overflow
    int result;

    // No overflow case
    bool overflow = __builtin_add_overflow(100, 200, &result);
    std::cout << "100 + 200 = " << result
              << (overflow ? " (OVERFLOW)" : " (ok)") << '\n';

    // Overflow case
    overflow = __builtin_add_overflow(INT_MAX, 1, &result);
    std::cout << "INT_MAX + 1 = " << result
              << (overflow ? " (OVERFLOW!)" : " (ok)") << '\n';

    // Negative overflow
    overflow = __builtin_add_overflow(INT_MIN, -1, &result);
    std::cout << "INT_MIN + (-1) = " << result
              << (overflow ? " (OVERFLOW!)" : " (ok)") << '\n';

    // Subtraction
    overflow = __builtin_sub_overflow(INT_MIN, 1, &result);
    std::cout << "INT_MIN - 1 = " << result
              << (overflow ? " (OVERFLOW!)" : " (ok)") << '\n';

    // Multiplication
    overflow = __builtin_mul_overflow(100000, 100000, &result);
    std::cout << "100000 * 100000 = " << result
              << (overflow ? " (OVERFLOW!)" : " (ok)") << '\n';

    // Cross-type: result type determines overflow detection
    int16_t small;
    overflow = __builtin_add_overflow(30000, 30000, &small);
    std::cout << "30000 + 30000 (int16) = " << small
              << (overflow ? " (OVERFLOW!)" : " (ok)") << '\n';
}
// Output:
//   100 + 200 = 300 (ok)
//   INT_MAX + 1 = -2147483648 (OVERFLOW!)
//   INT_MIN + (-1) = 2147483647 (OVERFLOW!)
//   INT_MIN - 1 = 2147483647 (OVERFLOW!)
//   100000 * 100000 = 1410065408 (OVERFLOW!)
//   30000 + 30000 (int16) = -5536 (OVERFLOW!)

```

### Q2: safe_add<T> wrapper

```cpp

#include <iostream>
#include <stdexcept>
#include <climits>
#include <cstdint>
#include <type_traits>

// Generic safe arithmetic with overflow checking
template<typename T>
T safe_add(T a, T b) {
    static_assert(std::is_integral_v<T>, "Integer types only");
    T result;
    if (__builtin_add_overflow(a, b, &result))
        throw std::overflow_error("Addition overflow");
    return result;
}

template<typename T>
T safe_sub(T a, T b) {
    T result;
    if (__builtin_sub_overflow(a, b, &result))
        throw std::overflow_error("Subtraction overflow");
    return result;
}

template<typename T>
T safe_mul(T a, T b) {
    T result;
    if (__builtin_mul_overflow(a, b, &result))
        throw std::overflow_error("Multiplication overflow");
    return result;
}

// Safe arithmetic class for chaining
template<typename T>
class SafeInt {
public:
    explicit SafeInt(T val) : val_(val) {}
    T value() const { return val_; }

    SafeInt operator+(SafeInt other) const {
        return SafeInt(safe_add(val_, other.val_));
    }
    SafeInt operator-(SafeInt other) const {
        return SafeInt(safe_sub(val_, other.val_));
    }
    SafeInt operator*(SafeInt other) const {
        return SafeInt(safe_mul(val_, other.val_));
    }

private:
    T val_;
};

int main() {
    // Function-style
    try {
        std::cout << safe_add(100, 200) << '\n';       // 300
        std::cout << safe_mul(50000, 50000) << '\n';    // throws!
    } catch (const std::overflow_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }

    // Class-style
    try {
        SafeInt<int32_t> a(INT_MAX - 10);
        SafeInt<int32_t> b(5);
        auto c = a + b;  // ok: INT_MAX - 5
        std::cout << "a + b = " << c.value() << '\n';

        auto d = a + SafeInt<int32_t>(20);  // throws!
        std::cout << "a + 20 = " << d.value() << '\n';
    } catch (const std::overflow_error& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }
}
// Output:
//   300
//   Caught: Multiplication overflow
//   a + b = 2147483642
//   Caught: Addition overflow

```

### Q3: Why signed overflow is UB

**The C++ standard says** ([basic.fundamental]/4): Signed integer arithmetic uses a mathematical model; overflow is undefined.

**Why does the standard do this?**

1. **Historical**: C was designed for diverse hardware (ones' complement, sign-magnitude, two's complement). Overflow behavior differed. UB made C portable.

2. **Optimization**: If the compiler knows overflow "can't happen", it can:
   - Assume `i + 1 > i` (loop termination)
   - Assume `x * 2 / 2 == x` (algebraic simplification)
   - Eliminate range checks
   - Vectorize loops with induction variables

3. **Contrast with unsigned**: Unsigned integers are defined to wrap modulo 2^N. This prevents some optimizations but gives predictable behavior.

```cpp

#include <iostream>
#include <climits>

int main() {
    // Unsigned: defined wrapping
    unsigned u = UINT_MAX;
    unsigned u2 = u + 1;  // defined: wraps to 0
    std::cout << "UINT_MAX + 1 = " << u2 << " (defined)\n";

    // Signed: UB
    int s = INT_MAX;
    // int s2 = s + 1;  // UB! Compiler may optimize away!

    // Practical consequence:
    // This loop may be optimized to infinite or removed entirely:
    //   for (int i = 0; i >= 0; ++i) { ... }
    // Compiler: "i starts at 0, always increases, so i >= 0 is always true"

    std::cout << "\nProtection strategies:\n";
    std::cout << "  1. __builtin_add_overflow (check before)\n";
    std::cout << "  2. -fsanitize=signed-integer-overflow (test time)\n";
    std::cout << "  3. -ftrapv (runtime trap)\n";
    std::cout << "  4. Use unsigned or wider types when safe\n";
    std::cout << "  5. Pre-condition checks: if (a > INT_MAX - b) ...\n";
}

```

---

## Notes

- Complementary to #733 (UBSan, constexpr detection, compiler exploitation examples).
- MSVC doesn't have `__builtin_*_overflow`. Use `_addcarry_u64` or SafeInt library.
- Microsoft's SafeInt library: `safeint.h` in Windows SDK.
- Rust equivalent: `.checked_add()`, `.wrapping_add()`, `.saturating_add()` — explicit choice.
- Performance: overflow builtins compile to test of CPU overflow flag (~0 overhead).
