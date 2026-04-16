# Use std::experimental::simd (C++ SIMD library) for portable vectorization

**Category:** Performance & CPU Architecture  
**Item:** #723  
**Reference:** <https://en.cppreference.com/w/cpp/experimental/simd>  

---

## Topic Overview

`std::experimental::simd` (Parallelism TS 2) provides a C++ abstraction over SIMD registers. It compiles to native SIMD instructions (AVX2, NEON, etc.) without platform-specific intrinsics.

```cpp

Raw intrinsics:                    stdx::simd:
  #ifdef __AVX2__                    stdx::native_simd<float> a, b;
  __m256 a, b, c;                    auto c = a + b;  // works everywhere!
  c = _mm256_add_ps(a, b);          // AVX2 -> vaddps ymm
  #elif __ARM_NEON                   // NEON -> fadd v0.4s
  float32x4_t a, b, c;              // SSE  -> addps xmm
  c = vaddq_f32(a, b);              // Scalar -> for loop
  #endif

```

| Header | Availability |
| --- | --- |
| `<experimental/simd>` | GCC 11+ (libstdc++) |
| [std-simd](https://github.com/VcDevel/std-simd) | Standalone, GCC/Clang |
| C++26 `<simd>` | Proposed for standardization |

---

## Self-Assessment

### Q1: Element-wise addition with `stdx::fixed_size_simd`

```cpp

#include <experimental/simd>
#include <iostream>
#include <array>

namespace stdx = std::experimental;

void add_arrays(const float* a, const float* b, float* c, int n) {
    // fixed_size_simd: exactly 8 lanes regardless of hardware
    using V = stdx::fixed_size_simd<float, 8>;
    constexpr int W = V::size();  // always 8

    int i = 0;
    for (; i + W <= n; i += W) {
        V va(a + i, stdx::element_aligned);  // load 8 floats
        V vb(b + i, stdx::element_aligned);
        V vc = va + vb;                      // SIMD add
        vc.copy_to(c + i, stdx::element_aligned);  // store
    }
    for (; i < n; ++i) c[i] = a[i] + b[i]; // scalar tail
}

void add_native(const float* a, const float* b, float* c, int n) {
    // native_simd: uses hardware-natural width (4 for SSE, 8 for AVX2)
    using V = stdx::native_simd<float>;
    constexpr int W = V::size();
    std::cout << "Native SIMD width: " << W << " floats\n";

    int i = 0;
    for (; i + W <= n; i += W) {
        V va(a + i, stdx::element_aligned);
        V vb(b + i, stdx::element_aligned);
        V vc = va + vb;
        vc.copy_to(c + i, stdx::element_aligned);
    }
    for (; i < n; ++i) c[i] = a[i] + b[i];
}

int main() {
    std::array<float, 16> a, b, c;
    for (int i = 0; i < 16; ++i) { a[i] = float(i); b[i] = 100.0f; }

    add_arrays(a.data(), b.data(), c.data(), 16);
    for (float v : c) std::cout << v << ' ';
    // 100 101 102 103 104 105 106 107 108 109 110 111 112 113 114 115
    std::cout << '\n';

    add_native(a.data(), b.data(), c.data(), 16);
}
// Compile: g++ -O2 -std=c++17 -mavx2 -o test test.cpp

```

### Q2: `simd_mask` for conditional branchless operations

```cpp

#include <experimental/simd>
#include <iostream>
#include <vector>

namespace stdx = std::experimental;

// Replace negative values with zero (branchless using mask)
void clamp_positive(float* data, int n) {
    using V = stdx::native_simd<float>;
    constexpr int W = V::size();
    V zero(0.0f);

    int i = 0;
    for (; i + W <= n; i += W) {
        V v(data + i, stdx::element_aligned);

        // simd_mask: boolean vector, one bit per lane
        typename V::mask_type negative = v < zero;
        // negative = {true, false, true, ...} (per-element comparison)

        // where(mask, v) = 0: conditionally assign zero to negative lanes
        stdx::where(negative, v) = zero;
        // NO branch! Compiles to:
        //   vcmpltps ymm1, ymm0, ymm2   (compare)
        //   vblendvps ymm0, ymm0, ymm2, ymm1  (conditional move)

        v.copy_to(data + i, stdx::element_aligned);
    }
    for (; i < n; ++i)
        if (data[i] < 0) data[i] = 0;
}

// Conditional multiply: double positive values, negate negative ones
void conditional_transform(float* data, int n) {
    using V = stdx::native_simd<float>;
    constexpr int W = V::size();

    int i = 0;
    for (; i + W <= n; i += W) {
        V v(data + i, stdx::element_aligned);
        V result = v;

        auto pos = v > V(0.0f);
        auto neg = v < V(0.0f);

        stdx::where(pos, result) = v * V(2.0f);   // double positives
        stdx::where(neg, result) = -v;              // negate negatives
        // zeros stay zero

        result.copy_to(data + i, stdx::element_aligned);
    }
    for (; i < n; ++i) {
        if (data[i] > 0) data[i] *= 2;
        else if (data[i] < 0) data[i] = -data[i];
    }
}

int main() {
    std::vector<float> data = {-3, 5, -1, 0, 7, -2, 4, -8, 1, -6};
    clamp_positive(data.data(), data.size());
    for (float v : data) std::cout << v << ' ';
    // 0 5 0 0 7 0 4 0 1 0
    std::cout << '\n';
}

```

### Q3: `stdx::simd` vs raw intrinsics

```cpp

#include <iostream>

int main() {
    std::cout << "stdx::simd vs raw intrinsics comparison:\n\n";
    std::cout << "+-------------------+-----------------------------+---------------------------+\n";
    std::cout << "| Feature           | stdx::simd                  | Raw intrinsics            |\n";
    std::cout << "+-------------------+-----------------------------+---------------------------+\n";
    std::cout << "| Portability       | x86/ARM/RISC-V (one code)   | Platform-specific         |\n";
    std::cout << "| Code readability  | a + b * c                   | _mm256_fmadd_ps(a,b,c)    |\n";
    std::cout << "| Type safety       | Strong (template types)     | Weak (all __m256)         |\n";
    std::cout << "| Masking           | where(mask, v) = x          | _mm256_blendv_ps(...)     |\n";
    std::cout << "| Performance       | Same (zero overhead)        | Same (manual control)     |\n";
    std::cout << "| Tail handling     | Still manual                | Still manual              |\n";
    std::cout << "| Learning curve    | Low (uses operators)        | High (intrinsic names)    |\n";
    std::cout << "| Maturity          | Experimental (TS 2)         | Stable, decades old       |\n";
    std::cout << "| Compiler support  | GCC 11+ (libstdc++)         | All major compilers       |\n";
    std::cout << "+-------------------+-----------------------------+---------------------------+\n";

    // Performance comparison (dot product, N=10M, AVX2):
    //   stdx::simd:     ~120ms (identical assembly!)
    //   Raw intrinsics:  ~120ms
    //   Scalar:           ~800ms
    //
    // The compiler optimizes stdx::simd to the SAME instructions as raw intrinsics.
    // Assembly proof (godbolt.org):
    //   stdx::native_simd<float> a + b  -> vaddps ymm0, ymm1, ymm2
    //   _mm256_add_ps(a, b)             -> vaddps ymm0, ymm1, ymm2
    //   Identical!
}

```

---

## Notes

- `stdx::native_simd<float>` adapts to hardware: 4 lanes (SSE), 8 (AVX2), 16 (AVX-512).
- `stdx::fixed_size_simd<float, N>` guarantees exact width regardless of hardware.
- GCC: `#include <experimental/simd>` with `-std=c++17`.
- `stdx::where(mask, vec) = val` is the branchless conditional — compiles to blend/select.
- Proposed for C++26 as `std::simd` (moving out of experimental).
