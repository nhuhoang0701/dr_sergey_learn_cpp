# Understand CPU branch prediction and how code structure affects it

**Category:** Performance & CPU Architecture  
**Item:** #623  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/likely>  

---

## Topic Overview

Modern CPUs speculatively execute instructions past branches, predicting which path will be taken. A misprediction flushes the pipeline (~15-20 cycles penalty).

```cpp

Branch Prediction Unit:

  Branch History Table (BHT)       Branch Target Buffer (BTB)
  +--------+--------+              +--------+-----------+
  | Branch | Pattern|              | Branch | Target    |
  |  addr  | (2-bit)|              |  addr  | address   |
  +--------+--------+              +--------+-----------+
  | 0x4010 | ST (11)|              | 0x4010 | 0x4020    |
  | 0x4030 | WN (01)|              | 0x4030 | 0x4050    |
  +--------+--------+              +--------+-----------+

  2-bit saturating counter:  ST=Strongly Taken, WT=Weakly Taken
                              WN=Weakly Not-taken, SN=Strongly Not-taken

```

| Predictor type | Description |
| --- | --- |
| Static | Forward branches: not-taken; Back branches: taken (loops) |
| 1-bit | Remembers last outcome |
| 2-bit saturating | Needs 2 mispredictions to change state |
| TAGE (modern) | Multiple history lengths, 95%+ accuracy |
| Loop predictor | Detects loop iteration counts |

---

## Self-Assessment

### Q1: How the CPU branch predictor works

```cpp

#include <iostream>

// The CPU branch predictor has several components:
//
// 1. BRANCH HISTORY TABLE (BHT) / Pattern History Table (PHT)
//    - Maps branch PC (program counter) to a 2-bit saturating counter
//    - States: SN(00) -> WN(01) -> WT(10) -> ST(11)
//    - Taken increments, not-taken decrements
//    - Prediction: counter >= 2 -> predict taken
//
// 2. BRANCH TARGET BUFFER (BTB)
//    - Cache mapping branch address -> target address
//    - For indirect jumps (virtual calls, switch), stores predicted target
//    - Misprediction on target costs ~20 cycles (flush + refetch)
//
// 3. GLOBAL HISTORY REGISTER (GHR)
//    - Shift register of last N branch outcomes (taken/not-taken)
//    - Correlates branches: "if (a) ... if (b)" where b depends on a
//    - XORed with branch PC to index BHT (gshare predictor)
//
// 4. TAGE PREDICTOR (modern: Intel Haswell+, AMD Zen+)
//    - Multiple tables with different history lengths
//    - Short history for simple patterns, long history for complex
//    - 95-97% accuracy on typical code

int main() {
    // Example: 2-bit counter evolution for a loop branch
    // "for (int i = 0; i < 100; ++i) { body; }"
    //
    // Iteration:  1    2    3   ... 99   100 (exit)
    // Actual:     T    T    T   ... T    NT
    // Counter:    01   10   11  ... 11   10  (predict T)
    //                                     ^- only 1 mispredict at exit!
    //
    // Total mispredictions for 100-iteration loop: 1 (at exit)
    // Cost: 1 * 15 cycles = 15 cycles overhead

    std::cout << "Branch prediction overview:\n";
    std::cout << "  BHT: direction prediction (taken/not-taken)\n";
    std::cout << "  BTB: target address prediction\n";
    std::cout << "  GHR: correlating branch history\n";
    std::cout << "  TAGE: multiple history lengths (modern CPUs)\n";
    std::cout << "  Mispredict penalty: ~15-20 cycles (pipeline flush)\n";
}

```

### Q2: Common-case-first ordering improves throughput

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>

enum class Type { Common, Rare1, Rare2, Rare3 };

// BAD: rare cases checked first
[[gnu::noinline]] int process_bad(Type t, int val) {
    if (t == Type::Rare3) return val * 7;    // 1% of cases
    if (t == Type::Rare2) return val * 5;    // 2% of cases
    if (t == Type::Rare1) return val * 3;    // 7% of cases
    return val * 2;                           // 90% of cases!
    // For 90% of inputs, we check 3 branches before reaching common case
    // All 3 branches are "not taken" -> predictor learns this, BUT
    // we waste 3 comparisons and add instruction cache pressure
}

// GOOD: common case first
[[gnu::noinline]] int process_good(Type t, int val) {
    if (t == Type::Common) [[likely]] return val * 2;  // 90%: first check!
    if (t == Type::Rare1) return val * 3;               // 7%
    if (t == Type::Rare2) return val * 5;               // 2%
    return val * 7;                                      // 1%
    // 90% of inputs take the FIRST branch -> 1 comparison
    // Predictor accuracy is same, but fewer instructions executed
}

int main() {
    constexpr int N = 10'000'000;
    std::mt19937 rng(42);
    std::vector<Type> types(N);
    for (auto& t : types) {
        int r = rng() % 100;
        if (r < 90) t = Type::Common;
        else if (r < 97) t = Type::Rare1;
        else if (r < 99) t = Type::Rare2;
        else t = Type::Rare3;
    }

    auto bench = [&](auto fn, const char* label) {
        volatile int sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) sum += fn(types[i], i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_bad,  "Rare-first ");
    bench(process_good, "Common-first");
    // Typical: Common-first is 10-20% faster
}

```

### Q3: Use `perf stat` to measure branch misses

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <random>

// Program to demonstrate branch misprediction with sorted vs unsorted data.
// The classic "branch prediction" benchmark.

int main() {
    constexpr int N = 100'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;

    // Unsorted: values are random -> 50% branch mispredict rate
    long long sum1 = 0;
    for (int rep = 0; rep < 1000; ++rep) {
        for (int x : data) {
            if (x >= 128)  // random: ~50% taken -> unpredictable!
                sum1 += x;
        }
    }

    // Sort the data: now values are ordered
    // First half: all < 128 (always not-taken)
    // Second half: all >= 128 (always taken)
    // Branch predictor accuracy: ~99.9%
    std::sort(data.begin(), data.end());

    long long sum2 = 0;
    for (int rep = 0; rep < 1000; ++rep) {
        for (int x : data) {
            if (x >= 128)  // sorted: predictor learns pattern -> ~0% mispredict
                sum2 += x;
        }
    }

    std::cout << "Sum unsorted: " << sum1 << '\n';
    std::cout << "Sum sorted:   " << sum2 << '\n';

    // Compile and measure:
    //   g++ -O2 -o branch branch.cpp
    //   perf stat -e branches,branch-misses ./branch
    //
    // UNSORTED:
    //   branches:         1,500,000,000
    //   branch-misses:      500,000,000  (33%!)
    //   Time: ~8 seconds
    //
    // SORTED:
    //   branches:         1,500,000,000
    //   branch-misses:          100,000  (<0.01%)
    //   Time: ~2 seconds  (4x faster!)
    //
    // Alternative: branchless version eliminates the issue entirely:
    //   sum += (x >= 128) * x;  // no branch, works on unsorted data
}

```

---

## Notes

- Modern CPUs (TAGE predictor) achieve 95%+ accuracy on real code.
- `[[likely]]` / `[[unlikely]]` (C++20) hint the compiler for code layout, not the hardware predictor.
- `perf stat -e branch-misses` is the key tool for diagnosing branch issues.
- Sorting input data can eliminate branch mispredictions (but adds sort cost).
- Virtual function calls use the BTB; polymorphic call sites with many types cause BTB misses.
- Switch statements: compiler may use jump table (indirect branch) or if-else chain depending on density.
