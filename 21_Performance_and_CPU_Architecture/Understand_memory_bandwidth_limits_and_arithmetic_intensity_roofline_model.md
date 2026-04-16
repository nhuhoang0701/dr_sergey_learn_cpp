# Understand memory bandwidth limits and arithmetic intensity (roofline model)

**Category:** Performance & CPU Architecture  
**Item:** #634  
**Reference:** <https://en.wikipedia.org/wiki/Roofline_model>  

---

## Topic Overview

The roofline model predicts performance as a function of arithmetic intensity (FLOP/byte). A kernel is either **compute-bound** or **memory-bound**.

```cpp

  Performance (GFLOP/s)
       ^
       |         _______________
       |        /               <- compute ceiling (peak FLOP/s)
       |       /
       |      /  <- memory bandwidth ceiling
       |     /
       |    /
       |   /
       +--/------------------------------> Arithmetic Intensity (FLOP/byte)
          ^                      ^
      memory-bound          compute-bound

  Ridge point = peak_GFLOPS / peak_bandwidth_GB_s

```

| Kernel | FLOP/byte | Bound |
| --- | --- | --- |
| Stream copy (`a[i]=b[i]`) | 0 | Memory |
| DAXPY (`a[i]=a[i]+s*b[i]`) | 2 FLOP / 16 bytes = 0.125 | Memory |
| Dot product | 2 FLOP / 8 bytes = 0.25 | Memory |
| Matrix multiply (NxN) | 2N³ / 3N²×8 = N/12 FLOP/byte | Compute (for large N) |

---

## Self-Assessment

### Q1: Arithmetic intensity of DGEMM vs stream copy

```cpp

#include <iostream>
#include <cmath>

int main() {
    // STREAM COPY: a[i] = b[i]
    //   FLOPs:  0 (just a copy)
    //   Bytes:  read 8 (double b) + write 8 (double a) = 16 bytes/element
    //   AI:     0 FLOP / 16 bytes = 0.0 FLOP/byte
    //   => ALWAYS memory-bound. Stream bandwidth is the limit.

    // DAXPY: a[i] = a[i] + s * b[i]
    //   FLOPs:  2 (multiply + add)
    //   Bytes:  read a (8) + read b (8) + write a (8) = 24 bytes
    //           (or 16 if write-allocate eliminated)
    //   AI:     2/24 = 0.083 or 2/16 = 0.125 FLOP/byte
    //   => Memory-bound on all modern CPUs

    // DGEMM (NxN): C = A * B
    //   FLOPs:  2*N^3 (N^3 multiply + N^3 add)
    //   Bytes:  3 * N^2 * 8 (read A, B, write C, each N^2 doubles)
    //   AI:     2*N^3 / (24*N^2) = N/12 FLOP/byte
    //   For N=1024: AI = 1024/12 = 85 FLOP/byte
    //   => Compute-bound for N > ~100!

    double peak_gflops = 200.0;    // typical Skylake AVX-512
    double bandwidth   = 50.0;     // GB/s DDR4 dual-channel
    double ridge_point = peak_gflops / bandwidth;  // FLOP/byte

    std::cout << "Peak compute: " << peak_gflops << " GFLOP/s\n";
    std::cout << "Peak bandwidth: " << bandwidth << " GB/s\n";
    std::cout << "Ridge point: " << ridge_point << " FLOP/byte\n";
    std::cout << '\n';

    auto classify = [&](const char* name, double ai) {
        double attainable = std::min(peak_gflops, bandwidth * ai);
        std::cout << name << ": AI=" << ai << " FLOP/byte -> "
                  << attainable << " GFLOP/s ("
                  << (ai < ridge_point ? "MEMORY-bound" : "COMPUTE-bound")
                  << ")\n";
    };

    classify("Stream copy   ", 0.0);
    classify("DAXPY         ", 0.125);
    classify("Dot product   ", 0.25);
    classify("SpMV          ", 0.5);
    classify("DGEMM (N=1024)", 85.0);
}

```

### Q2: Roofline model to classify compute-bound vs memory-bound

```cpp

#include <iostream>
#include <vector>
#include <chrono>

// Memory-bound kernel: stream triad  a[i] = b[i] + s * c[i]
void stream_triad(double* __restrict a, const double* __restrict b,
                  const double* __restrict c, double s, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = b[i] + s * c[i];
    }
    // AI = 2 FLOP / 24 bytes = 0.083
}

// Compute-bound kernel: many operations per element
void compute_heavy(double* __restrict a, const double* __restrict b, int n) {
    for (int i = 0; i < n; ++i) {
        double x = b[i];
        // 20 FLOPs per element:
        x = x*x + x; x = x*x + x; x = x*x + x; x = x*x + x;
        x = x*x + x; x = x*x + x; x = x*x + x; x = x*x + x;
        x = x*x + x; x = x*x + x;
        a[i] = x;
    }
    // AI = 20 FLOP / 16 bytes = 1.25
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<double> a(N), b(N, 1.1), c(N, 2.2);

    auto bench = [&](auto fn, const char* label, double flops_per_elem) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 10; ++r) fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        double sec = std::chrono::duration<double>(t1 - t0).count() / 10;
        double gflops = N * flops_per_elem / sec / 1e9;
        std::cout << label << ": " << gflops << " GFLOP/s (" << sec*1000 << " ms)\n";
    };

    bench([&]{ stream_triad(a.data(), b.data(), c.data(), 3.0, N); },
          "Stream triad (mem-bound) ", 2);
    bench([&]{ compute_heavy(a.data(), b.data(), N); },
          "Compute heavy (cpu-bound)", 20);

    // Stream triad: ~10 GFLOP/s (bounded by memory bandwidth)
    // Compute heavy: ~100 GFLOP/s (bounded by CPU FLOP throughput)
}

```

### Q3: Loop fusion to increase arithmetic intensity

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <cmath>

// UNFUSED: two separate loops, each memory-bound
void unfused(double* a, double* b, const double* c, int n) {
    // Loop 1: reads c, writes a (24 bytes/element)
    for (int i = 0; i < n; ++i)
        a[i] = c[i] * 2.0;

    // Loop 2: reads a, writes b (24 bytes/element)
    for (int i = 0; i < n; ++i)
        b[i] = std::sqrt(a[i]) + a[i];

    // Total: 3 FLOP / 48 bytes = 0.0625 FLOP/byte -> deeply memory-bound
    // Data traversed twice: c->a then a->b
    // a[] evicted from cache between loops if N is large!
}

// FUSED: single loop, better arithmetic intensity
void fused(double* a, double* b, const double* c, int n) {
    for (int i = 0; i < n; ++i) {
        a[i] = c[i] * 2.0;
        b[i] = std::sqrt(a[i]) + a[i];
    }
    // Total: 3 FLOP / 32 bytes = 0.094 FLOP/byte
    // But more importantly: a[i] is in register, NOT re-read from memory!
    // Effective: 3 FLOP / 24 bytes = 0.125 (c read once, a+b written once)
    // Data traversed once: much better cache behavior
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<double> a(N), b(N), c(N, 4.0);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int r = 0; r < 10; ++r) fn(a.data(), b.data(), c.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() / 10 << " ms\n";
    };

    bench(unfused, "Unfused (2 passes)");
    bench(fused,   "Fused (1 pass)    ");
    // Typical: fused is 30-50% faster (one memory pass vs two)
}

```

---

## Notes

- **Ridge point** = Peak GFLOP/s ÷ Peak bandwidth (GB/s). Below it: memory-bound.
- Most real-world code is memory-bound (AI < 1). Optimization = reduce data movement.
- Loop fusion, tiling, and AoS-to-SoA transforms increase arithmetic intensity.
- Intel tools: `likwid-perfctr` and `Intel Advisor` can generate roofline plots automatically.
- Stream benchmark (`STREAM Triad`) measures practical memory bandwidth.
