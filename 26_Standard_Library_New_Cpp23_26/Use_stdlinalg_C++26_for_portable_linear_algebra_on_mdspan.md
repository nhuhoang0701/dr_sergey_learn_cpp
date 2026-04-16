# Use std::linalg (C++26) for portable linear algebra on mdspan

**Category:** Standard Library — New in C++23/26  
**Item:** #582  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/numeric/linalg>  

---

## Topic Overview

`std::linalg` (C++26, `<linalg>`) provides **standard linear algebra operations** that work on `std::mdspan` views. It replaces the need for external libraries (Eigen, BLAS wrappers) for common operations and can map to optimized BLAS implementations under the hood.

### Key Operations

| Function                          | BLAS Level | Description                      |
| --- | --- | --- |
| `linalg::dot(x, y)`              | 1          | Dot product of two vectors       |
| `linalg::vector_norm2(x)`        | 1          | Euclidean norm (L2)              |
| `linalg::scale(alpha, x)`        | 1          | x = alpha * x                    |
| `linalg::matrix_vector_product(A, x, y)` | 2  | y = A * x                       |
| `linalg::matrix_product(A, B, C)`| 3          | C = A * B                        |
| `linalg::scaled(alpha, A)`       | —          | Lazy scaling view (no temporary) |
| `linalg::transposed(A)`          | —          | Lazy transpose view              |

### Design Principle

```cpp

Raw data (array/vector)  →  mdspan (view with layout)  →  linalg (operations)
     ↑                            ↑                            ↑
  Owns memory             No ownership, zero copy        Works on any mdspan

```

---

## Self-Assessment

### Q1: Compute a matrix-vector product using std::linalg::matrix_vector_product with mdspan inputs

**Answer:**

```cpp

#include <linalg>   // C++26
#include <mdspan>   // C++23
#include <vector>
#include <iostream>
#include <array>

int main() {
    // Matrix A (2x3):
    // | 1  2  3 |
    // | 4  5  6 |
    std::array<double, 6> a_data = {1, 2, 3, 4, 5, 6};
    std::mdspan A(a_data.data(), 2, 3);  // 2 rows, 3 cols (row-major default)

    // Vector x (3):
    std::array<double, 3> x_data = {1, 0, 2};
    std::mdspan x(x_data.data(), 3);

    // Result vector y (2):
    std::array<double, 2> y_data = {};
    std::mdspan y(y_data.data(), 2);

    // y = A * x
    // y[0] = 1*1 + 2*0 + 3*2 = 7
    // y[1] = 4*1 + 5*0 + 6*2 = 16
    std::linalg::matrix_vector_product(A, x, y);

    std::cout << "y = A * x:\n";
    for (size_t i = 0; i < 2; ++i)
        std::cout << "  y[" << i << "] = " << y_data[i] << '\n';
    // y[0] = 7
    // y[1] = 16

    // ═══════════ With scaling: y = 2 * A * x ═══════════
    std::array<double, 2> y2_data = {};
    std::mdspan y2(y2_data.data(), 2);
    std::linalg::matrix_vector_product(std::linalg::scaled(2.0, A), x, y2);
    std::cout << "y2 = 2*A*x: " << y2_data[0] << ", " << y2_data[1] << '\n';
    // 14, 32

    // ═══════════ With transposed: y = A^T * z ═══════════
    // A^T is 3x2, so z must be size 2, result is size 3
    std::array<double, 2> z_data = {1, 1};
    std::mdspan z(z_data.data(), 2);
    std::array<double, 3> y3_data = {};
    std::mdspan y3(y3_data.data(), 3);

    std::linalg::matrix_vector_product(std::linalg::transposed(A), z, y3);
    // A^T[0] = {1,4}, A^T[1] = {2,5}, A^T[2] = {3,6}
    // y3 = {5, 7, 9}
    std::cout << "y3 = A^T*z: ";
    for (size_t i = 0; i < 3; ++i) std::cout << y3_data[i] << ' ';
    std::cout << '\n';
}

```

### Q2: Use std::linalg::scaled to scale a matrix in-place without a temporary

**Answer:**

```cpp

#include <linalg>
#include <mdspan>
#include <array>
#include <iostream>

int main() {
    // ═══════════ scaled() creates a LAZY VIEW — no allocation ═══════════
    std::array<double, 4> data = {1.0, 2.0, 3.0, 4.0};
    std::mdspan vec(data.data(), 4);

    // scaled() does NOT modify data — it returns a view that multiplies on access
    auto twice = std::linalg::scaled(2.0, vec);
    // twice[i] computes 2.0 * vec[i] on the fly — no temporary array created

    std::cout << "Original: ";
    for (size_t i = 0; i < 4; ++i) std::cout << vec[i] << ' ';
    std::cout << '\n';  // 1 2 3 4

    std::cout << "Scaled view: ";
    for (size_t i = 0; i < 4; ++i) std::cout << twice[i] << ' ';
    std::cout << '\n';  // 2 4 6 8

    // Original is unchanged:
    std::cout << "Original still: ";
    for (size_t i = 0; i < 4; ++i) std::cout << vec[i] << ' ';
    std::cout << '\n';  // 1 2 3 4

    // ═══════════ Use in expressions: y = alpha*A*x ═══════════
    // Without scaled(): must allocate temp = alpha*A, then compute temp*x
    // With scaled(): linalg::matrix_vector_product(scaled(alpha, A), x, y)
    //   → applies alpha during the multiply, no temporary matrix

    std::array<double, 4> mat_data = {1, 2, 3, 4};  // 2x2 matrix
    std::mdspan mat(mat_data.data(), 2, 2);
    std::array<double, 2> x_data = {1, 1};
    std::mdspan x(x_data.data(), 2);
    std::array<double, 2> y_data = {};
    std::mdspan y(y_data.data(), 2);

    // y = 3.0 * mat * x — NO temporary for "3.0 * mat"
    std::linalg::matrix_vector_product(std::linalg::scaled(3.0, mat), x, y);
    std::cout << "y = 3*M*x: " << y_data[0] << ", " << y_data[1] << '\n';
    // mat*x = {3, 7}, so 3*mat*x = {9, 21}

    // ═══════════ In-place scaling with scale() ═══════════
    // To actually modify the data:
    std::linalg::scale(5.0, vec);  // vec[i] *= 5.0 for all i
    std::cout << "After in-place scale(5): ";
    for (size_t i = 0; i < 4; ++i) std::cout << vec[i] << ' ';
    std::cout << '\n';  // 5 10 15 20
}

```

### Q3: Explain how linalg maps to optimised BLAS routines on platforms that provide them

**Answer:**

```cpp

┌───────────────────────────────────────────────────────────────────────┐
│                    std::linalg Architecture                          │
├───────────────────────────────────────────────────────────────────────┤
│  User code:  linalg::matrix_vector_product(A, x, y)                 │
│                          │                                           │
│                   ┌──────┴──────┐                                    │
│                   │ C++ stdlib  │  ← standard interface              │
│                   └──────┬──────┘                                    │
│              ┌───────────┼───────────┐                               │
│              ▼           ▼           ▼                                │
│       ┌──────────┐ ┌──────────┐ ┌──────────┐                        │
│       │ Fallback │ │ OpenBLAS │ │  MKL     │  ← vendor backends     │
│       │ (generic)│ │ (dgemv)  │ │ (cblas_  │                        │
│       │ portable │ │ optimized│ │  dgemv)  │                        │
│       └──────────┘ └──────────┘ └──────────┘                        │
│                                                                      │
│  BLAS Levels:                                                        │
│  Level 1: dot, nrm2, scale, axpy      ← vector-vector  O(n)        │
│  Level 2: gemv, trsv, ger             ← matrix-vector  O(n²)       │
│  Level 3: gemm, trsm, syrk            ← matrix-matrix  O(n³)       │
└───────────────────────────────────────────────────────────────────────┘

```

```cpp

// How the mapping works:

// std::linalg function         → BLAS routine
// ─────────────────────────────────────────────
// linalg::dot(x, y)           → cblas_ddot (Level 1)
// linalg::vector_norm2(x)     → cblas_dnrm2 (Level 1)
// linalg::scale(alpha, x)     → cblas_dscal (Level 1)
// linalg::matrix_vector_product(A, x, y) → cblas_dgemv (Level 2)
// linalg::matrix_product(A, B, C)        → cblas_dgemm (Level 3)

// The implementation is allowed to:
// 1. Call native BLAS (OpenBLAS, MKL, Accelerate) for hardware optimization
// 2. Fall back to a portable C++ implementation on platforms without BLAS
// 3. Use execution policies for parallelism:
//    linalg::matrix_product(std::execution::par, A, B, C);
//    → may use BLAS multi-threaded or std::par implementation

// Key insight: mdspan layout tags enable optimization:
// - layout_right (C row-major)  → maps directly to cblas with CblasRowMajor
// - layout_left (Fortran col-major) → maps directly to BLAS native order
// - Custom layouts → fall back to generic implementation

// Performance example (1000x1000 matrix multiply):
// Generic C++ fallback:  ~2000 ms
// OpenBLAS (dgemm):      ~15 ms    (>100x faster)
// Intel MKL (dgemm):     ~10 ms    (>200x faster)
// → Same std::linalg code, vastly different performance based on backend

```

---

## Notes

- `std::linalg` is C++26; reference implementation: `https://github.com/kokkos/stdBLAS`
- All operations take `mdspan` views — no ownership, zero-copy, works on existing data
- `scaled()` and `transposed()` are **lazy views** — no temporary allocation
- Execution policies (`std::execution::par`) enable multi-threaded execution
- For small matrices (< 4x4), hand-written code may beat BLAS due to call overhead
