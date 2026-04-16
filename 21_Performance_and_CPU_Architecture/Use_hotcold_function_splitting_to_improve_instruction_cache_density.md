# Use hot/cold function splitting to improve instruction cache density

**Category:** Performance & CPU Architecture  
**Item:** #628  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Common-Function-Attributes.html>  

---

## Topic Overview

Hot/cold splitting separates frequently-executed code from rarely-executed code (error handling, logging) into different text sections, improving instruction cache density for the hot path.

```cpp

Without splitting:                    With splitting:
.text:                                .text.hot:
  [process entry]                       [process entry]
  [hot loop body]                       [hot loop body]
  [error handler - 200 bytes]           [process exit]
  [hot loop body continued]           .text.unlikely:
  [logging - 500 bytes]                 [error handler]
  [process exit]                        [logging]
  Total hot path: fragmented            Total hot path: compact!
  icache lines: scattered               icache lines: contiguous

```

| Technique | Mechanism |
| --- | --- |
| `[[gnu::cold]]` function | Entire function placed in `.text.unlikely` |
| `noinline` + cold | Prevent cold code from bloating hot functions |
| PGO (`-fprofile-use`) | Compiler automatically splits based on profile |
| `-freorder-functions` | GCC groups hot functions together |

---

## Self-Assessment

### Q1: `[[gnu::cold]]` on error paths for icache improvement

```cpp

#include <iostream>
#include <vector>
#include <stdexcept>
#include <chrono>

// WITHOUT splitting: error handling code is inlined into hot function
[[gnu::hot]] int process_no_split(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < 0) {
            // This error handling code is INSIDE the hot function!
            // It takes ~200 bytes of icache even though it rarely executes.
            std::cerr << "Error at index " << i
                      << ": negative value " << data[i] << '\n';
            throw std::runtime_error("negative value");
        }
        sum += data[i];
    }
    return sum;
}

// WITH splitting: error handling extracted to cold function
[[gnu::cold]] [[noreturn]]
void report_error(int idx, int val) {
    std::cerr << "Error at index " << idx
              << ": negative value " << val << '\n';
    throw std::runtime_error("negative value");
}

[[gnu::hot]] int process_split(const int* data, int n) {
    int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < 0) [[unlikely]] {
            report_error(i, data[i]);  // cold call, out of hot section
        }
        sum += data[i];
    }
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N, 42);

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile int s = 0;
        for (int r = 0; r < 100; ++r) s = fn(data.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_no_split, "No split ");
    bench(process_split,    "Split    ");
    // Split version: hot loop is more compact -> fewer icache misses
    // Improvement: ~5-15% for tight loops with error handling
}

```

### Q2: `noinline` on cold paths prevents hot-function bloating

```cpp

#include <iostream>
#include <string>

// Problem: compiler may INLINE cold helper into hot function,
// which bloats the hot function's code size.

// GOOD: [[gnu::noinline]] prevents inlining of cold code
[[gnu::cold]] [[gnu::noinline]]
void log_debug(const char* msg, int val) {
    // This code is 100+ bytes of instructions.
    // Without noinline, compiler might inline it into hot function!
    std::string s = "DEBUG: ";
    s += msg;
    s += " value=";
    s += std::to_string(val);
    std::cerr << s << '\n';
}

[[gnu::cold]] [[gnu::noinline]]
void handle_boundary(int idx, int n) {
    std::cerr << "Boundary: idx=" << idx << " n=" << n << '\n';
}

[[gnu::hot]] int hot_compute(const int* data, int n) {
    int result = 0;
    for (int i = 0; i < n; ++i) {
        result += data[i] * data[i];

        // Without noinline: log_debug's code would be copied HERE,
        // expanding the loop body by ~100 bytes.
        // With noinline: just a `call` instruction (5 bytes).
        if (result > 1'000'000) [[unlikely]] {
            log_debug("overflow", result);
        }
        if (i == n - 1) [[unlikely]] {
            handle_boundary(i, n);
        }
    }
    return result;
}

// To verify:
//   g++ -O2 -S test.cpp
//   Look at hot_compute in assembly:
//   - With noinline: loop body is ~20 instructions (compact)
//   - Without noinline: loop body is ~80 instructions (bloated)
//   The compact version uses fewer icache lines.

int main() {
    int data[] = {1, 2, 3, 4, 5};
    std::cout << hot_compute(data, 5) << '\n';  // 55
}

```

### Q3: PGO-based automatic hot/cold section grouping

```cpp

#include <iostream>

// Profile-Guided Optimization (PGO) automates hot/cold splitting:
//
// Step 1: Instrumented build
//   g++ -O2 -fprofile-generate -o app app.cpp
//
// Step 2: Run with representative workload
//   ./app <typical_input>
//   // Generates .gcda profile data files
//
// Step 3: Optimized build using profile
//   g++ -O2 -fprofile-use -o app app.cpp
//
// What the compiler does with profile data:
//
// 1. FUNCTION REORDERING (-freorder-functions):
//    Hot functions grouped in .text.hot
//    Cold functions in .text.unlikely
//    Result: hot functions are contiguous in memory -> icache friendly
//
// 2. BASIC BLOCK REORDERING (-freorder-blocks):
//    Hot basic blocks placed contiguously (fall-through path)
//    Cold basic blocks (error handling) moved to end of function
//    Result: hot path has no jumps -> better icache and prefetcher
//
// 3. FUNCTION SPLITTING (-freorder-blocks-and-partition):
//    A SINGLE function is split: hot part in .text.hot, cold part in .text.unlikely
//    Example: process() has hot loop + cold error handler
//    -> Hot loop in .text.hot, error handler in .text.unlikely
//
// Linker integration:
//   -ffunction-sections -fdata-sections: each function gets its own section
//   Linker script or --symbol-ordering-file orders sections by hotness
//   Gold/LLD: --section-ordering-file=hot_order.txt

int main() {
    std::cout << "PGO hot/cold splitting workflow:\n";
    std::cout << "  1. g++ -O2 -fprofile-generate -o app app.cpp\n";
    std::cout << "  2. ./app <representative_workload>\n";
    std::cout << "  3. g++ -O2 -fprofile-use -o app app.cpp\n";
    std::cout << "\nCompiler optimizations enabled by PGO:\n";
    std::cout << "  -freorder-functions: group hot functions\n";
    std::cout << "  -freorder-blocks: reorder basic blocks\n";
    std::cout << "  -freorder-blocks-and-partition: split functions\n";
    std::cout << "\nTypical improvement: 10-20% for large codebases\n";
}

```

---

## Notes

- `[[gnu::cold]]` + `[[gnu::noinline]]` is the manual equivalent of PGO splitting.
- PGO is superior because it uses actual execution data, not programmer guesses.
- BOLT (Binary Optimization and Layout Tool) can reorder functions post-link for even better results.
- For large codebases (Chrome, Firefox), PGO + BOLT gives 5-15% overall speedup.
- `perf stat -e L1-icache-load-misses` measures instruction cache pressure.
