# Understand cache-oblivious algorithms and recursive tiling

**Category:** Performance & CPU Architecture  
**Item:** #541  
**Reference:** <https://en.wikipedia.org/wiki/Cache-oblivious_algorithm>  

---

## Topic Overview

Cache-oblivious algorithms achieve optimal cache performance on all levels of the memory hierarchy without knowing cache sizes. They use recursive divide-and-conquer until subproblems fit in any cache.

```cpp

Naive triple-loop matrix multiply:      Recursive (cache-oblivious):
  for i: 0..N                             split(A, B, C) {
    for j: 0..N                              if small enough: naive multiply
      for k: 0..N                            else:
        C[i][j] += A[i][k] * B[k][j]          split into 8 sub-problems
                                               recurse on each
  Accesses B column-wise:                    }
  stride = N * sizeof(double)             Each sub-problem eventually fits
  -> cache misses on every access!        in L1 -> minimal cache misses

```

| Algorithm | Cache complexity | Notes |
| --- | --- | --- |
| Naive matmul | O(N³/B) | B = cache line, terrible for B[k][j] |
| Cache-oblivious matmul | O(N³/(B√M)) | M = cache size, optimal |
| Naive merge sort | O(N/B · log N) | Streaming access |
| Cache-oblivious merge sort | O(N/B · log(N/M)) | Optimal |

---

## Self-Assessment

### Q1: Why recursive matrix multiplication has better cache behavior

```cpp

#include <iostream>
#include <vector>
#include <chrono>

using Matrix = std::vector<std::vector<double>>;

// Naive O(N^3) matrix multiply
void naive_multiply(const Matrix& A, const Matrix& B, Matrix& C, int N) {
    for (int i = 0; i < N; ++i)
        for (int j = 0; j < N; ++j)
            for (int k = 0; k < N; ++k)
                C[i][j] += A[i][k] * B[k][j];  // B accessed column-wise!
    // B[k][j]: stride = N doubles between consecutive k values
    // For N=1024: stride = 8KB -> every access misses L1 cache!
}

// Cache-oblivious recursive multiply
void recursive_multiply(const Matrix& A, const Matrix& B, Matrix& C,
                         int ar, int ac, int br, int bc, int cr, int cc, int n) {
    if (n <= 64) {  // Base case: fits in L1 cache (~32KB)
        for (int i = 0; i < n; ++i)
            for (int j = 0; j < n; ++j)
                for (int k = 0; k < n; ++k)
                    C[cr+i][cc+j] += A[ar+i][ac+k] * B[br+k][bc+j];
        return;
    }
    int h = n / 2;
    // C11 = A11*B11 + A12*B21
    recursive_multiply(A, B, C, ar,   ac,   br,   bc,   cr,   cc,   h);
    recursive_multiply(A, B, C, ar,   ac+h, br+h, bc,   cr,   cc,   h);
    // C12 = A11*B12 + A12*B22
    recursive_multiply(A, B, C, ar,   ac,   br,   bc+h, cr,   cc+h, h);
    recursive_multiply(A, B, C, ar,   ac+h, br+h, bc+h, cr,   cc+h, h);
    // C21 = A21*B11 + A22*B21
    recursive_multiply(A, B, C, ar+h, ac,   br,   bc,   cr+h, cc,   h);
    recursive_multiply(A, B, C, ar+h, ac+h, br+h, bc,   cr+h, cc,   h);
    // C22 = A21*B12 + A22*B22
    recursive_multiply(A, B, C, ar+h, ac,   br,   bc+h, cr+h, cc+h, h);
    recursive_multiply(A, B, C, ar+h, ac+h, br+h, bc+h, cr+h, cc+h, h);
}

int main() {
    const int N = 512;
    Matrix A(N, std::vector<double>(N, 1.0));
    Matrix B(N, std::vector<double>(N, 1.0));
    Matrix C1(N, std::vector<double>(N, 0.0));
    Matrix C2(N, std::vector<double>(N, 0.0));

    auto bench = [](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ naive_multiply(A, B, C1, N); }, "Naive    ");
    bench([&]{ recursive_multiply(A, B, C2, 0,0,0,0,0,0, N); }, "Recursive");
    // N=512: Naive ~400ms, Recursive ~150ms (2.5x faster)
    // The difference grows with N (more cache misses in naive)
}

```

### Q2: Cache-oblivious merge sort

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <chrono>
#include <random>

// Cache-oblivious merge sort: recursively divides until
// subproblem fits in cache (any cache level) automatically.
void merge(int* arr, int* tmp, int lo, int mid, int hi) {
    int i = lo, j = mid, k = lo;
    while (i < mid && j < hi)
        tmp[k++] = (arr[i] <= arr[j]) ? arr[i++] : arr[j++];
    while (i < mid) tmp[k++] = arr[i++];
    while (j < hi)  tmp[k++] = arr[j++];
    std::copy(tmp + lo, tmp + hi, arr + lo);
}

void merge_sort(int* arr, int* tmp, int lo, int hi) {
    if (hi - lo <= 1) return;
    // No cache-size parameter! Just keep dividing.
    // Eventually subproblems fit in L1 (32KB = ~4000 ints)
    int mid = lo + (hi - lo) / 2;
    merge_sort(arr, tmp, lo, mid);
    merge_sort(arr, tmp, mid, hi);
    merge(arr, tmp, lo, mid, hi);
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N), tmp(N);
    std::mt19937 rng(42);
    std::generate(data.begin(), data.end(), rng);

    auto data2 = data; // copy for std::sort comparison

    auto t0 = std::chrono::high_resolution_clock::now();
    merge_sort(data.data(), tmp.data(), 0, N);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms1 = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);

    auto t2 = std::chrono::high_resolution_clock::now();
    std::sort(data2.begin(), data2.end());
    auto t3 = std::chrono::high_resolution_clock::now();
    auto ms2 = std::chrono::duration_cast<std::chrono::milliseconds>(t3 - t2);

    std::cout << "Cache-oblivious merge sort: " << ms1.count() << " ms\n";
    std::cout << "std::sort (introsort):      " << ms2.count() << " ms\n";
    std::cout << "Sorted correctly: " << std::is_sorted(data.begin(), data.end()) << '\n';
}

```

### Q3: Performance cliff at L1, L2, L3 boundaries

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>

int main() {
    // Measure random access latency at different working set sizes
    // to reveal cache hierarchy boundaries.

    // Typical Intel cache sizes:
    // L1: 32KB,  L2: 256KB,  L3: 8MB
    std::vector<int> sizes = {
        4*1024, 8*1024, 16*1024, 32*1024,       // L1 region
        64*1024, 128*1024, 256*1024,              // L2 region
        512*1024, 1024*1024, 4*1024*1024,         // L3 region
        8*1024*1024, 16*1024*1024, 64*1024*1024   // Main memory
    };

    for (int bytes : sizes) {
        int n = bytes / sizeof(int);
        std::vector<int> arr(n);
        // Create a random permutation (pointer chase)
        std::iota(arr.begin(), arr.end(), 0);
        std::mt19937 rng(42);
        std::shuffle(arr.begin(), arr.end(), rng);

        // Pointer chase: arr[i] -> arr[arr[i]] -> ...
        volatile int idx = 0;
        constexpr int ITERS = 10'000'000;

        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < ITERS; ++i) {
            idx = arr[idx % n];
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count()
                    / static_cast<double>(ITERS);

        std::cout << (bytes / 1024) << " KB: " << ns << " ns/access\n";
    }
    // Expected output (approximate):
    //    4 KB: ~1.5 ns    (L1 hit)
    //   32 KB: ~1.5 ns    (L1 boundary)
    //   64 KB: ~4 ns      (L2 hit)     <-- cliff!
    //  256 KB: ~4 ns      (L2 boundary)
    //  512 KB: ~12 ns     (L3 hit)     <-- cliff!
    //    8 MB: ~12 ns     (L3 boundary)
    //   16 MB: ~60 ns     (DRAM)       <-- cliff!
}

```

---

## Notes

- Cache-oblivious algorithms don't need cache-size parameters — they adapt automatically.
- Key insight: recursive subdivision ensures subproblems fit in *every* cache level.
- Real-world use: FFTW (fastest FFT) uses cache-oblivious recursive algorithms.
- Cache complexity notation: O(f(N, M, B)) where M = cache size, B = cache line size.
- For practical code, cache-aware blocking (tuned to L1/L2 size) often beats cache-oblivious by 10-20%.
