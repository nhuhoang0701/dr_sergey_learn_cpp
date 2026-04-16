# Use loop vectorization hints and verify SIMD code generation

**Category:** Performance & CPU Architecture  
**Item:** #626  
**Standard:** C++11  
**Reference:** <https://llvm.org/docs/Vectorizers.html>  

---

## Topic Overview

Auto-vectorization converts scalar loops into SIMD instructions. The compiler needs hints (pragmas, annotations) when it can't prove safety, and you should always verify the generated assembly.

```cpp

Scalar loop:                    Vectorized loop (AVX2):
  for (int i=0; i<N; ++i)        for (int i=0; i<N; i+=8)
    a[i] = b[i] + c[i];            vmovups ymm0, [b+i*4]
  // 1 add per cycle                vaddps  ymm0, ymm0, [c+i*4]
                                    vmovups [a+i*4], ymm0
                                  // 8 adds per cycle (8x speedup)

```

| Hint / Pragma | Compiler | Effect |
| --- | --- | --- |
| `#pragma GCC ivdep` | GCC | Ignore vector dependencies |
| `#pragma clang loop vectorize(enable)` | Clang | Force vectorization attempt |
| `#pragma omp simd` | GCC/Clang/MSVC | Portable, force SIMD |
| `__restrict` | All | No pointer aliasing |
| `__builtin_assume_aligned` | GCC/Clang | Aligned access (no scalar peel) |

---

## Self-Assessment

### Q1: Use `#pragma GCC ivdep` and `__builtin_assume_aligned` for vectorization

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <cstring>

// Problem: compiler can't prove a and b don't alias.
// Without hints, the compiler generates scalar code + alias check.
void add_no_hint(float* a, const float* b, const float* c, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c[i];
    // Compiler MAY vectorize but adds runtime alias check:
    //   cmp rdi, rsi      ; do a and b overlap?
    //   jb  .scalar_loop  ; if yes, fall back to scalar
}

// Solution 1: __restrict tells compiler pointers don't alias
void add_restrict(float* __restrict a, const float* __restrict b,
                  const float* __restrict c, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c[i];
    // Always vectorized, no alias check needed
}

// Solution 2: #pragma GCC ivdep ignores ALL dependencies
void add_ivdep(float* a, const float* b, const float* c, int n) {
#pragma GCC ivdep
    for (int i = 0; i < n; ++i)
        a[i] = b[i] + c[i];
    // Vectorized. WARNING: UB if arrays actually overlap!
}

// Solution 3: Aligned access (avoids scalar peeling loop)
void add_aligned(float* a, const float* b, const float* c, int n) {
    float* aa = static_cast<float*>(__builtin_assume_aligned(a, 32));
    const float* bb = static_cast<const float*>(__builtin_assume_aligned(b, 32));
    const float* cc = static_cast<const float*>(__builtin_assume_aligned(c, 32));
    for (int i = 0; i < n; ++i)
        aa[i] = bb[i] + cc[i];
    // vmovaps instead of vmovups (aligned = no peeling loop)
}

int main() {
    constexpr int N = 10'000'000;
    // Allocate 32-byte aligned memory for AVX
    alignas(32) static float a[N], b[N], c[N];
    std::memset(b, 1, sizeof(b));
    std::memset(c, 2, sizeof(c));

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 100; ++r) fn(a, b, c, N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(add_no_hint,  "No hint  ");
    bench(add_restrict, "Restrict ");
    bench(add_ivdep,    "ivdep    ");
    bench(add_aligned,  "Aligned  ");
}

```

### Q2: Verify SIMD code generation on Compiler Explorer

```cpp

#include <iostream>

// How to verify vectorization:
//
// Method 1: Compiler Explorer (godbolt.org)
//   Paste code, add -O2 -mavx2, look for:
//     vmovups -> unaligned 256-bit load
//     vmovaps -> aligned 256-bit load
//     vaddps  -> 8-float add
//     vfmadd  -> fused multiply-add
//
// Method 2: GCC vectorization report
//   g++ -O2 -mavx2 -ftree-vectorizer-verbose=2 test.cpp
//   or (modern):
//   g++ -O2 -mavx2 -fopt-info-vec-all test.cpp
//   Output: "test.cpp:10: note: LOOP VECTORIZED"
//   or:     "test.cpp:15: note: not vectorized: data ref analysis failed"
//
// Method 3: Clang vectorization report
//   clang++ -O2 -mavx2 -Rpass=loop-vectorize test.cpp
//   Output: "vectorized loop (vectorization width: 8, interleaved count: 4)"
//   For failures:
//   clang++ -O2 -Rpass-missed=loop-vectorize test.cpp

// Example: this vectorizes:
void vectorizes(float* __restrict a, const float* __restrict b, int n) {
    for (int i = 0; i < n; ++i)
        a[i] = b[i] * 2.0f + 1.0f;
    // Assembly (AVX2):
    //   vmovups  ymm0, [rsi + rcx*4]      ; load 8 floats from b
    //   vmulps   ymm0, ymm0, ymm1         ; multiply by 2.0
    //   vaddps   ymm0, ymm0, ymm2         ; add 1.0
    //   vmovups  [rdi + rcx*4], ymm0       ; store 8 floats to a
}

// Example: this does NOT vectorize:
float no_vectorize(const float* a, int n) {
    float sum = 0;
    for (int i = 0; i < n; ++i)
        sum += a[i];
    // Without -ffast-math: NOT vectorized!
    // Reason: FP addition is not associative.
    //   sum += a[0]; sum += a[1]; ...  (must be sequential)
    // With -ffast-math: vectorized using horizontal add
    return sum;
}

// Fix: manual reduction for FP sum without -ffast-math
float vectorized_sum(const float* a, int n) {
    float s0 = 0, s1 = 0, s2 = 0, s3 = 0;
    int i = 0;
    for (; i + 3 < n; i += 4) {
        s0 += a[i];   s1 += a[i+1];
        s2 += a[i+2]; s3 += a[i+3];
    }
    for (; i < n; ++i) s0 += a[i];
    return s0 + s1 + s2 + s3;
    // 4 independent accumulator chains -> vectorizable!
}

int main() {
    float data[] = {1,2,3,4,5,6,7,8};
    std::cout << vectorized_sum(data, 8) << '\n'; // 36
}

```

### Q3: Loop-carried dependencies that prevent vectorization

```cpp

#include <iostream>
#include <vector>

// Loop-carried dependency: iteration i depends on iteration i-1
// The compiler CANNOT vectorize these loops.

// PROBLEM 1: Prefix sum (each element depends on previous)
void prefix_sum_bad(float* a, int n) {
    for (int i = 1; i < n; ++i)
        a[i] += a[i-1];  // a[i] depends on a[i-1] -> sequential!
    // NOT vectorizable: must complete i-1 before starting i
}

// FIX: Use parallel prefix sum (Blelloch scan)
void prefix_sum_parallel(float* a, int n) {
    // Up-sweep (reduce)
    for (int d = 1; d < n; d *= 2)
        for (int i = d-1; i+d < n; i += 2*d)
            a[i+d] += a[i];  // inner loop is independent -> vectorizable
    // Down-sweep (distribute) also vectorizable
}

// PROBLEM 2: Recurrence relation
void recurrence_bad(float* a, float c, int n) {
    for (int i = 1; i < n; ++i)
        a[i] = a[i-1] * c + a[i];  // depends on a[i-1] -> sequential!
}

// FIX: Unroll and express as matrix operation
// a[i]   = c * a[i-1] + a[i]
// a[i+1] = c * a[i]   + a[i+1] = c^2 * a[i-1] + c*a[i] + a[i+1]
// After enough unrolling, each group is independent.

// PROBLEM 3: Linked list traversal
struct Node { Node* next; int val; };
int sum_list_bad(Node* head) {
    int sum = 0;
    for (Node* p = head; p; p = p->next)  // p->next is loop-carried
        sum += p->val;
    return sum;  // NOT vectorizable: pointer chasing
}

// FIX: Convert to array-based structure
int sum_array(const std::vector<int>& vals) {
    int sum = 0;
    for (int v : vals) sum += v;  // contiguous, vectorizable!
    return sum;
}

int main() {
    // To check: g++ -O2 -mavx2 -fopt-info-vec-missed test.cpp
    // Output shows WHICH loops failed and WHY:
    //   note: not vectorized: complex access pattern
    //   note: not vectorized: unsupported use in stmt
    //   note: read-write dependency detected
    std::cout << "Always verify with -fopt-info-vec-all\n";
}

```

---

## Notes

- `#pragma omp simd` is most portable (C++11 with OpenMP 4.0+, works on MSVC/GCC/Clang).
- `-fopt-info-vec-all` (GCC) or `-Rpass=loop-vectorize` (Clang) are essential verification tools.
- Common vectorization blockers: pointer aliasing, FP associativity, loop-carried deps, function calls.
- `__restrict` is the most impactful single annotation for enabling vectorization.
- Always compile with `-O2 -mavx2` (or `-march=native`) for auto-vectorization.
