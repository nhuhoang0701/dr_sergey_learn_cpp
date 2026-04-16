# Understand signed integer overflow UB and why it enables optimizations

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/ub>  

---

## Topic Overview

### Signed Overflow Is Undefined Behavior

In C++, **signed integer overflow** is undefined behavior. The standard says nothing about what happens when a computation on signed integers exceeds the range of the type — the compiler is free to assume it **never happens**.

```cpp

#include <climits>

int x = INT_MAX;
int y = x + 1;  // UB! Result is not defined.
                 // NOT guaranteed to wrap to INT_MIN.

```

**Unsigned integers**, by contrast, are defined to wrap modulo 2^N:

```cpp

unsigned u = UINT_MAX;
unsigned v = u + 1;  // Defined: v == 0 (wraps around)

```

### Why the Standard Made It UB

The decision was deliberate and has roots in hardware diversity:

- Some CPUs use **two's complement** (most modern CPUs)
- Some historical CPUs used **one's complement** or **sign-magnitude**
- On trap-based architectures, overflow triggers a hardware exception

By leaving signed overflow undefined, the standard allows C++ to run efficiently on all architectures without mandating a specific overflow behavior.

### How Compilers Exploit Signed Overflow UB

Because the compiler can assume signed overflow never occurs, it can make powerful optimizations:

#### 1. Loop bound optimization

```cpp

// The compiler ASSUMES i + 1 > i (no overflow), so this loop terminates
for (int i = 0; i < n; ++i) {
    process(i);
}
// Generated code: simple compare-and-branch, no overflow check

```

If signed overflow *could* wrap, `i` might wrap from `INT_MAX` to `INT_MIN`, making `i < n` true forever — an infinite loop. The compiler uses the UB guarantee to prove the loop terminates.

#### 2. Simplifying comparisons

```cpp

bool always_positive(int x) {
    return (x + 1) > x;  // Compiler: "always true" (overflow can't happen)
}
// Compiled to: return true;

```

#### 3. Eliminating redundant checks

```cpp

int f(int x) {
    if (x > 0 && x + 1 > 0) {  // x + 1 > 0 is always true if x > 0
        return x + 1;            // (because overflow can't happen)
    }
    return 0;
}
// Compiler optimizes to: if (x > 0) return x + 1;

```

#### 4. Induction variable strength reduction

```cpp

void fill_multiples(int* arr, int n) {
    for (int i = 0; i < n; ++i) {
        arr[i] = i * 4;  // Compiler converts to: prev += 4
    }
}
// Multiplication per iteration → single addition per iteration
// Only valid if i*4 never overflows (signed UB assumption)

```

#### 5. Range propagation

```cpp

int clamp_positive(int x) {
    if (x < 1) x = 1;
    // Compiler now knows x >= 1
    // Also deduces x + 1 >= 2 (no overflow assumption)
    return x + 1;  // Result is always >= 2
}

```

### Real-World Consequences — Vanishing Security Checks

```cpp

// Simplified from a real Linux kernel vulnerability
void check_size(int offset, int len) {
    if (offset + len < offset) {  // Overflow check
        return;  // Reject
    }
    memcpy(dst, src + offset, len);
}

```

With `-O2`, the compiler sees `offset + len < offset`. Since signed overflow is UB, it assumes `offset + len >= offset` is always true. The compiler **removes the entire if-block**. The security check vanishes. The fix is to use unsigned arithmetic or compiler builtins.

### How to Write Overflow-Safe Code

#### Option 1: Pre-check before the operation

```cpp

#include <climits>

bool would_overflow_add(int a, int b) {
    if (b > 0 && a > INT_MAX - b) return true;
    if (b < 0 && a < INT_MIN - b) return true;
    return false;
}

```

#### Option 2: Use compiler builtins (GCC/Clang)

```cpp

int safe_add(int a, int b) {
    int result;
    if (__builtin_add_overflow(a, b, &result)) {
        return INT_MAX;  // or handle error
    }
    return result;
}
// Also: __builtin_sub_overflow, __builtin_mul_overflow

```

#### Option 3: Use wider types

```cpp

int safe_multiply(int a, int b) {
    int64_t wide = static_cast<int64_t>(a) * b;
    if (wide > INT_MAX || wide < INT_MIN) {
        return INT_MAX;
    }
    return static_cast<int>(wide);
}

```

#### Option 4: `-fwrapv` compiler flag

```bash

g++ -fwrapv -O2 main.cpp

```

Makes signed overflow defined as two's complement wrapping. Disables the optimizations above.

#### Option 5: UBSan for detection

```bash

g++ -fsanitize=undefined -O2 main.cpp
# Runtime: "signed integer overflow: 2147483647 + 1"

```

---

## Self-Assessment

### Q1: Why can't the compiler optimize unsigned overflow the same way

Unsigned overflow is **defined behavior** — it wraps modulo 2^N. The compiler must produce the wrapped result, so:

- `u + 1 > u` is **not** always true for unsigned (false when `u == UINT_MAX`)
- The compiler cannot eliminate overflow checks on unsigned types
- Loop optimizations based on "always increasing" assumptions don't apply

This is why signed integers sometimes generate **faster** code — the compiler has more freedom.

### Q2: Show a case where signed overflow UB causes a security vulnerability

```cpp

void allocate_buffer(int count) {
    int total = count * sizeof(Element);  // Overflow if count is huge!
    // Compiler assumes no overflow, so total > 0 if count > 0
    char* buf = new char[total];  // Allocates SMALL buffer
    for (int i = 0; i < count; i++) {
        read_element(buf + i * sizeof(Element));  // Heap overflow!
    }
}

// Fix: use size_t and validate
void allocate_buffer_safe(size_t count) {
    if (count > SIZE_MAX / sizeof(Element)) return;
    size_t total = count * sizeof(Element);
    auto buf = std::make_unique<char[]>(total);
}

```

### Q3: What does `-fwrapv` do and when should you use it

`-fwrapv` makes signed integer overflow **defined** as two's complement wrapping:

- `INT_MAX + 1` is guaranteed to equal `INT_MIN`
- The compiler can no longer assume `a + 1 > a` for signed types
- Security-related overflow checks will not be optimized away
- Some loop optimizations are disabled → potentially slower code

Use it when porting legacy code that relies on wrapping, or in security-critical code where overflow checks must not be removed.

---

## Notes

- C++20 mandates two's complement **representation**, but overflow is **still UB**
- UBSan is the best tool to find signed overflow in a codebase
- `std::midpoint(a, b)` (C++20) computes averages without overflow
- `std::cmp_less`, `std::cmp_greater` (C++20) do safe signed/unsigned comparisons
- GCC/Clang `-ftrapv` generates traps on overflow (slower than UBSan, but catches in production)
