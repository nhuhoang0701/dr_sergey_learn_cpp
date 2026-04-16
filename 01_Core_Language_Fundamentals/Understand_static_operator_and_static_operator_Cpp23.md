# Understand static operator() and static operator[] (C++23)

**Category:** Core Language Fundamentals  
**Standard:** C++23  
**Reference:** <https://wg21.link/P1169>  

---

## Topic Overview

C++23 allows `operator()` and `operator[]` to be `static`. This eliminates the implicit `this` pointer when the callable is stateless, enabling better optimization.

### Static operator()

```cpp

#include <algorithm>
#include <vector>
#include <iostream>

// Before C++23: stateless lambda still has a this pointer
auto old_cmp = [](int a, int b) { return a < b; };
// sizeof(old_cmp) == 1, but operator() takes hidden this* parameter

// C++23: static lambda — no this pointer at all
auto new_cmp = [](int a, int b) static { return a < b; };
// Truly stateless — no this pointer in generated code

struct Comparator {
    static bool operator()(int a, int b) { return a < b; }
    // C++23: static operator() is now valid
};

int main() {
    std::vector v{3, 1, 4, 1, 5};
    std::sort(v.begin(), v.end(), new_cmp);
    for (int x : v) std::cout << x << " "; // 1 1 3 4 5
}

```

### Static operator[]

```cpp

#include <array>
#include <iostream>

struct Constants {
    static constexpr std::array<double, 3> values = {3.14, 2.71, 1.41};

    // C++23: static operator[] for table-like access
    static constexpr double operator[](size_t i) { return values[i]; }
};

int main() {
    // Use without creating an instance:
    std::cout << Constants{}[0] << "\n";  // 3.14

    // C++23: multi-dimensional subscript
    struct Matrix {
        double data[3][3] = {};
        double& operator[](size_t r, size_t c) { return data[r][c]; }
        // Could also be static if no instance state needed
    };
    Matrix m;
    m[1, 2] = 42.0;  // C++23 multidimensional subscript
}

```

---

## Self-Assessment

### Q1: Why does removing the `this` pointer matter

Without `this`, the compiler doesn't pass a hidden first argument. For small functions used as predicates (sort comparators, hash functions), this saves a register and can enable better inlining. When used with `qsort`-style C APIs, a static `operator()` can be converted to a plain function pointer.

### Q2: When must a lambda be `static`

A lambda can be `static` only if it has no captures. `[x](int a) static { ... }` is a compile error because `x` requires a `this` pointer to access. Only `[](...) static { ... }` is valid.

### Q3: What's the benefit for C interop

```cpp

// A captureless static lambda can decay to a function pointer:
auto cmp = [](const void* a, const void* b) static -> int {
    return *(const int*)a - *(const int*)b;
};
// cmp is convertible to int(*)(const void*, const void*)
// Can be passed to qsort() directly

// Before C++23: captureless lambdas could already convert to function pointers
// But the static keyword makes the intent explicit and may optimize better

```

---

## Notes

- Static `operator()` is a minor optimization but documents intent clearly.
- Static `operator[]` enables multi-dimensional subscript (C++23): `m[i, j]`.
- Particularly useful for transparent comparators and hash functions.
- GCC 13+, Clang 16+, MSVC 19.34+ support static operator().
