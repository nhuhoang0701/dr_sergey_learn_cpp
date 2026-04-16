# Understand loop unrolling and control it with pragmas and attributes

**Category:** Performance & CPU Architecture  
**Item:** #547  
**Reference:** <https://llvm.org/docs/LoopTerminology.html>  

---

## Topic Overview

Loop unrolling replicates the loop body multiple times per iteration, reducing loop overhead (compare, branch) and enabling the CPU to fill execution pipelines.

```cpp

Original loop:                Unrolled x4:
  for (i=0; i<N; ++i)          for (i=0; i<N; i+=4) {
    a[i] = b[i] + c[i];          a[i]   = b[i]   + c[i];
                                  a[i+1] = b[i+1] + c[i+1];
                                  a[i+2] = b[i+2] + c[i+2];
                                  a[i+3] = b[i+3] + c[i+3];
  1 branch per iteration        }
                                1 branch per 4 iterations

```

| Benefit | Drawback |
| --- | --- |
| Less loop overhead | Larger code size (icache pressure) |
| Better ILP (instruction-level parallelism) | Register pressure may increase |
| Enables SIMD vectorization | Cleanup code for remainder (N%unroll) |
| Helps prefetcher | Diminishing returns past 4-8x |

---

## Self-Assessment

### Q1: Manual unrolling with `#pragma`

```cpp

#include <iostream>
#include <vector>
#include <chrono>

void add_no_pragma(float* __restrict a, const float* __restrict b,
                   const float* __restrict c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
}

void add_unroll4(float* __restrict a, const float* __restrict b,
                 const float* __restrict c, int n) {
    #pragma GCC unroll 4
    // Clang: #pragma unroll(4)
    // MSVC:  #pragma loop(unroll, 4)
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
    // Compiler generates equivalent of:
    //   for (i=0; i<n; i+=4) {
    //     a[i]=b[i]+c[i]; a[i+1]=b[i+1]+c[i+1];
    //     a[i+2]=b[i+2]+c[i+2]; a[i+3]=b[i+3]+c[i+3];
    //   }
    //   // plus cleanup for n%4 remainder
}

void add_unroll_full(float* __restrict a, const float* __restrict b,
                     const float* __restrict c, int n) {
    // For known-small loops:
    #pragma GCC unroll 16
    for (int i = 0; i < 16; ++i) {
        a[i] = b[i] + c[i];
    }
    // Fully unrolled: NO loop at all, just 16 straight-line instructions
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<float> a(N), b(N, 1.0f), c(N, 2.0f);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 100; ++r) fn(a.data(), b.data(), c.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() / 100 << " us\n";
    };

    bench(add_no_pragma, "No pragma  ");
    bench(add_unroll4,   "Unroll x4  ");
    // Compile: g++ -O2 -march=native -o test test.cpp
}

```

### Q2: When unrolling helps vs hurts

```cpp

#include <iostream>
#include <vector>
#include <chrono>

// HELPS: pipeline fill, ILP, reduced branch overhead
void sum_unrolled(const float* data, int n, float& result) {
    float s0 = 0, s1 = 0, s2 = 0, s3 = 0; // 4 independent accumulators
    int i = 0;
    for (; i + 3 < n; i += 4) {
        s0 += data[i];     // These 4 adds are INDEPENDENT
        s1 += data[i+1];   // CPU can execute them in parallel
        s2 += data[i+2];   // (out-of-order engine, multiple add units)
        s3 += data[i+3];
    }
    for (; i < n; ++i) s0 += data[i]; // cleanup
    result = s0 + s1 + s2 + s3;
}

// Without unrolling: serial dependency chain
void sum_serial(const float* data, int n, float& result) {
    float s = 0;
    for (int i = 0; i < n; ++i) {
        s += data[i];  // Each add depends on previous result!
        // Latency: ~3-4 cycles per add (FP pipeline)
        // Throughput limited to 1 add every 3-4 cycles
    }
    result = s;
}

// HURTS: over-unrolling causes icache pressure
void sum_overunrolled(const float* data, int n, float& result) {
    #pragma GCC unroll 64  // Way too much!
    float s = 0;
    for (int i = 0; i < n; ++i) s += data[i];
    result = s;
    // 64x unrolled: loop body doesn't fit in L1 icache (32KB)
    // Result: icache misses SLOW DOWN the code!
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<float> data(N, 1.0f);
    float r;

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn(data.data(), N, r);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(sum_serial,       "Serial (no unroll) ");
    bench(sum_unrolled,     "Unrolled x4        ");
    bench(sum_overunrolled, "Over-unrolled x64  ");
    // Serial:  ~100ms (latency-bound: 1 add/3 cycles)
    // x4:      ~25ms  (4 independent chains: 4 adds/3 cycles)
    // x64:     ~30ms  (icache pressure counteracts the benefit)
}

```

### Q3: Verify unrolling with `-funroll-loops` and assembly inspection

```cpp

#include <iostream>

// Compile with different flags and compare assembly:
//
//   g++ -O2 -S test.cpp -o no_unroll.s
//   g++ -O2 -funroll-loops -S test.cpp -o unrolled.s
//
// Or use Compiler Explorer (godbolt.org):
//   Flags: -O2 vs -O2 -funroll-loops

void array_add(float* __restrict a, const float* __restrict b,
               const float* __restrict c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + c[i];
    }
}

// WITHOUT -funroll-loops (g++ -O2):
// .L3:
//   movss  xmm0, [rsi + rax*4]      ; load b[i]
//   addss  xmm0, [rdx + rax*4]      ; add c[i]
//   movss  [rdi + rax*4], xmm0       ; store a[i]
//   inc    rax                        ; i++
//   cmp    rax, rcx                   ; i < n?
//   jl     .L3                        ; branch (1 per iteration)
//
// WITH -funroll-loops (g++ -O2 -funroll-loops):
// .L3:
//   movss  xmm0, [rsi + rax*4]      ; iter 0
//   addss  xmm0, [rdx + rax*4]
//   movss  [rdi + rax*4], xmm0
//   movss  xmm0, [rsi + rax*4 + 4]  ; iter 1
//   addss  xmm0, [rdx + rax*4 + 4]
//   movss  [rdi + rax*4 + 4], xmm0
//   ...                               ; iter 2, 3
//   add    rax, 4
//   cmp    rax, rcx
//   jl     .L3                        ; 1 branch per 4 iterations
//
// With -O3 or -O2 -ftree-vectorize -march=native:
//   Uses SIMD: movups, addps (4 floats at once!)
//   Combined with unrolling: 16 floats per loop iteration

int main() {
    float a[16], b[16], c[16];
    for (int i = 0; i < 16; ++i) { b[i] = i; c[i] = i * 2; }
    array_add(a, b, c, 16);
    for (int i = 0; i < 4; ++i)
        std::cout << a[i] << ' ';  // 0 3 6 9
    std::cout << '\n';
}

```

---

## Notes

- `#pragma GCC unroll N` (GCC 8+) / `#pragma unroll(N)` (Clang) / `#pragma loop(unroll, N)` (MSVC).
- `-funroll-loops` enables automatic unrolling; `-O3` includes it for GCC.
- Sweet spot: unroll 4-8x for most loops. Beyond that, icache pressure dominates.
- Unrolling is most beneficial for loops with independent iterations (no loop-carried dependency).
- For FP reductions, use multiple accumulators to break dependency chains.
