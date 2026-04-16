# Understand __restrict semantics and UB implications

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/restrict> (C99)  

---

## Topic Overview

### What Is `restrict`

`restrict` is a C99 keyword (not part of standard C++) that tells the compiler: "this pointer is the **only** way to access the memory it points to." This promise enables aggressive optimizations, particularly vectorization and instruction reordering.

C++ compilers provide it as a vendor extension:

- GCC/Clang: `__restrict__` or `__restrict`
- MSVC: `__restrict`

```cpp

// C99 (also works in C++ with extension keyword)
void add_arrays(float* __restrict out,
                const float* __restrict a,
                const float* __restrict b,
                int n) {
    for (int i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];
    }
}

```

### Why `restrict` Matters for Performance

Without `restrict`, the compiler must assume that `out`, `a`, and `b` might overlap (alias). This prevents optimizations:

```cpp

// WITHOUT restrict — compiler must assume aliasing
void add_arrays(float* out, const float* a, const float* b, int n) {
    for (int i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];
        // Compiler: "writing to out[i] might change a[i+1] or b[i+1]"
        // Must reload a[i+1] and b[i+1] from memory each iteration
    }
}

```

```cpp

// WITH restrict — compiler knows no aliasing
void add_arrays(float* __restrict out,
                const float* __restrict a,
                const float* __restrict b, int n) {
    for (int i = 0; i < n; ++i) {
        out[i] = a[i] + b[i];
        // Compiler: "out doesn't alias a or b"
        // Can vectorize: load 4 floats at once, add, store
    }
}
// Generated code: SIMD instructions (movaps, addps, movaps) processing 4 elements/iteration

```

### The UB: Violating the `restrict` Contract

If you promise `restrict` but the pointers **do** alias, the behavior is undefined:

```cpp

void scale(float* __restrict out, const float* __restrict in, int n) {
    for (int i = 0; i < n; ++i)
        out[i] = in[i] * 2.0f;
}

float arr[100];
// UB! out and in point to the same memory
scale(arr, arr, 100);  // Compiler may vectorize incorrectly

// Also UB: partial overlap
scale(arr + 1, arr, 99);  // in[0..98] overlaps with out[0..98]

```

The compiler may:

- Reorder reads and writes (producing wrong results)
- Vectorize with SIMD (reading 4 elements before any are written)
- Hoist loads out of the loop
- Generate completely wrong results silently

### `restrict` vs Strict Aliasing

They solve related but different problems:

| Concept | What it means | UB if violated |
| --- | --- | --- |
| **Strict aliasing** | Two pointers of **different types** don't alias | Yes |
| **`restrict`** | Two pointers of the **same type** don't alias | Yes |

Strict aliasing is a language rule (always in effect). `restrict` is an **opt-in annotation** the programmer adds.

```cpp

// Strict aliasing handles this (different types):
float* f = ...;
int* i = ...;  // Compiler assumes f and i don't alias

// restrict handles this (same type, different objects):
float* __restrict a = ...;
float* __restrict b = ...;  // Programmer promises a and b don't alias

```

### Safe Usage Patterns

```cpp

// Pattern 1: Function parameters (most common and safest)
void transform(float* __restrict dst,
               const float* __restrict src,
               size_t n);

// Pattern 2: Local restrict pointers from a struct
struct Matrix {
    float* data;
    int rows, cols;
};

void multiply(Matrix& C, const Matrix& A, const Matrix& B) {
    float* __restrict c = C.data;
    const float* __restrict a = A.data;
    const float* __restrict b = B.data;
    // c, a, b are promised not to alias each other
    for (int i = 0; i < A.rows; ++i)
        for (int j = 0; j < B.cols; ++j) {
            float sum = 0;
            for (int k = 0; k < A.cols; ++k)
                sum += a[i * A.cols + k] * b[k * B.cols + j];
            c[i * C.cols + j] = sum;
        }
}

```

### Checking If `restrict` Helped

Use Compiler Explorer (Godbolt) to compare generated assembly:

```bash

# GCC: optimization report
g++ -O3 -fopt-info-vec-optimized -S code.cpp
# "loop vectorized using 32 byte vectors"

# Clang: optimization remarks
clang++ -O3 -Rpass=loop-vectorize -S code.cpp

```

Without `restrict`, you'll often see a "can't vectorize: possible aliasing" remark. With `restrict`, the loop vectorizes.

---

## Self-Assessment

### Q1: Why isn't `restrict` in the C++ standard

Several reasons:

1. **Semantics complexity**: In C++, pointers come from references, `this`, iterators, and smart pointers. Defining `restrict` for all these contexts is hard.
2. **Template interactions**: What does `restrict` mean on a `T*` when `T` might be a reference?
3. **Existing alternative**: C++ has strict aliasing rules via type punning restrictions, which solve part of the problem.
4. **WG21 hasn't reached consensus**: Several proposals have been made (P1296, P0298) but none adopted.

All major compilers support `__restrict` as an extension, so it's usable in practice despite not being standard.

### Q2: Show the performance impact of restrict on a SAXPY operation

```cpp

// SAXPY: y = a * x + y
// Without restrict — compiler may not vectorize
void saxpy(float* y, const float* x, float a, int n) {
    for (int i = 0; i < n; ++i)
        y[i] = a * x[i] + y[i];  // y and x might alias!
}

// With restrict — compiler vectorizes freely
void saxpy_fast(float* __restrict y, const float* __restrict x,
                float a, int n) {
    for (int i = 0; i < n; ++i)
        y[i] = a * x[i] + y[i];
}
// Generated: vmulps + vaddps operating on 8 floats/iteration (AVX)
// Speedup: typically 4-8x on modern CPUs

```

### Q3: What happens if you violate the `restrict` contract

Undefined behavior. The compiler may have reordered or vectorized the loop assuming no aliasing. Results include:

- **Silent wrong results** — the most common and dangerous outcome
- The program appears to work on one compiler/optimization level but fails on another
- Sanitizers (`-fsanitize=address`) won't catch it — this is a semantic error, not a memory error
- Only careful code review or formal verification can catch restrict violations

---

## Notes

- Use `restrict` on function parameters in hot numerical loops — this is where the payoff is
- Never use `restrict` unless you can guarantee no aliasing for the lifetime of the scope
- `std::ranges` algorithms and `std::execution` parallel policies can sometimes give the compiler similar freedom
- In C++23, `[[assume(ptr1 != ptr2)]]` can express specific non-aliasing assumptions
- Fortran arrays are `restrict` by default — this is a key reason Fortran can generate faster numerical code than C/C++
