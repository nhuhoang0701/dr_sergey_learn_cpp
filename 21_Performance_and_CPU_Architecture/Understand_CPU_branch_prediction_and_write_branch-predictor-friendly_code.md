# Understand CPU branch prediction and write branch-predictor-friendly code

**Category:** Performance & CPU Architecture  
**Item:** #539  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/likely>  

---

## Topic Overview

Writing branch-predictor-friendly code means structuring branches so the common path is predictable and fall-through, minimizing costly mispredictions.

| Technique | Description | Benefit |
| --- | --- | --- |
| Common case fall-through | Put likely path as the "not-taken" branch | Predict-not-taken is default for forward branches |
| `[[likely]]` / `[[unlikely]]` | C++20 branch hints | Compiler arranges code layout |
| `__builtin_expect` | GCC/Clang branch hint | Pre-C++20 equivalent |
| Branchless arithmetic | Replace branch with `cmov` or math | Eliminates misprediction entirely |
| Loop peeling | Handle boundary cases outside loop | Removes branches from hot loop |

```cpp

Branchy hot loop:                   Branchless equivalent:
  for each x:                        for each x:
    if (x >= 128)   <- 50% miss        sum += (x >= 128) * x;
      sum += x;                         ^-- no branch, just multiply

```

---

## Self-Assessment

### Q1: Mispredict penalty on out-of-order CPUs

```cpp

#include <iostream>

// Modern out-of-order (OoO) CPUs execute instructions speculatively:
//
// 1. FETCH STAGE: Branch predictor guesses direction before decode
// 2. SPECULATIVE EXECUTION: CPU executes ~100+ instructions past the branch
// 3. ON CORRECT: Speculative results committed (free!)
// 4. ON MISPREDICT:
//    a. All speculative work is DISCARDED
//    b. Pipeline is FLUSHED (14-20 stages emptied)
//    c. Fetch restarts from correct target
//    d. ~10-20 cycles of no useful work ("pipeline bubble")
//
// Why 10-20 cycles specifically?
//   - Pipeline depth determines the cost
//   - Skylake: 14-19 stages -> ~15 cycle penalty
//   - The OoO engine can partially hide this by doing other work,
//     but in tight loops there's nothing else to do
//
// Practical impact:
//   CPU at 4 GHz, 4-wide issue:
//   1 mispredict = 15 cycles * 4 IPC = 60 instruction slots wasted
//   If loop body is 10 instructions:
//     Perfect prediction: 10/4 = 2.5 cycles per iteration
//     50% mispredict:     2.5 + 0.5 * 15 = 10 cycles per iteration (4x slower!)

int main() {
    std::cout << "Mispredict penalty breakdown:\n";
    std::cout << "  1. Detect mispredict: ~3 cycles (at execute stage)\n";
    std::cout << "  2. Flush pipeline:    ~2 cycles\n";
    std::cout << "  3. Refetch correct:   ~5 cycles (icache hit)\n";
    std::cout << "  4. Refill pipeline:   ~5 cycles\n";
    std::cout << "  Total: ~15 cycles on Skylake\n";
}

```

### Q2: Fall-through path optimization

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// "Fall-through" = the path taken when the branch is NOT taken.
// Static prediction: forward branches are predicted NOT-taken.
// Compilers arrange code so the likely path is the fall-through.

// BAD: common case requires a branch (jump over error handling)
[[gnu::noinline]] int process_bad(int x) {
    if (x < 0) {          // rare: error case
        return -1;         // <- fall-through (but x<0 is rare!)
    }
    return x * 2;          // <- common case requires jump
}

// GOOD: common case is fall-through
[[gnu::noinline]] int process_good(int x) {
    if (x >= 0) [[likely]] {  // common: 99% of cases
        return x * 2;         // <- fall-through (common case!)
    }
    return -1;                 // <- rare error path
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = (rng() % 100 == 0) ? -1 : rng() % 1000;
    // 1% negative, 99% positive

    auto bench = [&](auto fn, const char* label) {
        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int x : data) sum += fn(x);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(process_bad,  "Error-first  ");
    bench(process_good, "Common-first ");
    // Difference is small here (predictor learns), but code layout
    // affects icache: common-first keeps hot code contiguous.
}

```

### Q3: Branchless arithmetic eliminates misprediction

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// The ultimate branch-predictor-friendly code: NO BRANCHES.

// Branchy version:
[[gnu::noinline]] long long sum_branchy(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] >= 128)     // branch: ~50% mispredict for random data
            sum += data[i];
    }
    return sum;
}

// Branchless version 1: multiply by boolean
[[gnu::noinline]] long long sum_branchless_mul(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        sum += static_cast<long long>(data[i] >= 128) * data[i];
        // (data[i] >= 128) is 0 or 1
        // Multiply: either 0 * data[i] = 0, or 1 * data[i] = data[i]
        // No branch instruction generated!
    }
    return sum;
}

// Branchless version 2: bitwise mask
[[gnu::noinline]] long long sum_branchless_mask(const int* data, int n) {
    long long sum = 0;
    for (int i = 0; i < n; ++i) {
        int mask = -(data[i] >= 128);  // 0x00000000 or 0xFFFFFFFF
        sum += data[i] & mask;
    }
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 256;  // random: worst case for branch predictor

    auto bench = [&](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile auto s = fn(data.data(), N);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms (sum=" << s << ")\n";
    };

    bench(sum_branchy,         "Branchy        ");
    bench(sum_branchless_mul,  "Branchless *   ");
    bench(sum_branchless_mask, "Branchless mask");
    // Typical (random data, N=10M):
    // Branchy:         ~25 ms  (50% branch mispredict)
    // Branchless *:    ~8 ms   (no branches, auto-vectorized)
    // Branchless mask: ~8 ms   (same, different approach)
    // 3x speedup from eliminating branches!
}

```

---

## Notes

- `[[likely]]` / `[[unlikely]]` (C++20) affect compiler code layout, not hardware prediction.
- `__builtin_expect(expr, val)` is the pre-C++20 GCC/Clang equivalent.
- The compiler at `-O2`+ often converts simple branches to cmov automatically.
- For complex conditions, manually writing branchless code can beat the compiler.
- Profile first with `perf stat -e branch-misses` to confirm misprediction is the bottleneck.
