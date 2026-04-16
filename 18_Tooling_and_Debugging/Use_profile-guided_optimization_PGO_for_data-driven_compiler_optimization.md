# Use profile-guided optimization (PGO) for data-driven compiler optimization

**Category:** Tooling & Debugging  
**Item:** #413  
**Reference:** <https://clang.llvm.org/docs/UsersManual.html#profile-guided-optimization>  

---

## Topic Overview

PGO uses runtime profiling data to guide compiler optimizations. The compiler makes better decisions about inlining, branch layout, and code placement based on real execution patterns.

```cpp

PGO workflow (3 steps):

1. INSTRUMENT           2. PROFILE              3. OPTIMIZE

   g++ -fprofile-generate   ./app < test_data      g++ -fprofile-use
   -> app (instrumented)    -> *.gcda files         -> app (optimized)
   (slower, collects data)  (run representative     (uses profile to
                             workload)               optimize hot paths)

```

| PGO Benefit | How |
| --- | --- |
| Better branch prediction | Compiler lays out likely branches first |
| Better inlining | Inline hot functions, don't inline cold ones |
| Better register allocation | Prioritize hot loops |
| Code layout | Group hot code together (fewer icache misses) |

---

## Self-Assessment

### Q1: Run the three PGO steps

```cpp

// workload.cpp — a CPU-bound program to optimize with PGO
#include <vector>
#include <algorithm>
#include <numeric>
#include <iostream>
#include <random>

// This function has unpredictable branches without profile data:
int classify(int x) {
    if (x < 10)    return 0;   // 5% of inputs (rare)
    if (x < 100)   return 1;   // 15% of inputs
    if (x < 1000)  return 2;   // 30% of inputs
    return 3;                   // 50% of inputs (most common)
}

long long process(const std::vector<int>& data) {
    long long result = 0;
    for (int x : data) {
        int category = classify(x);
        switch (category) {
            case 0: result += x;     break;
            case 1: result += x * 2; break;
            case 2: result += x * 3; break;
            case 3: result += x * 4; break;  // most common path
        }
    }
    return result;
}

int main() {
    std::mt19937 rng(42);
    std::uniform_int_distribution<int> dist(1, 5000);
    std::vector<int> data(10'000'000);
    std::generate(data.begin(), data.end(), [&]{ return dist(rng); });

    auto result = process(data);
    std::cout << "Result: " << result << '\n';
}

```

```bash

# === GCC PGO ===
# Step 1: Instrument build
g++ -O2 -std=c++20 -fprofile-generate -o app_instr workload.cpp

# Step 2: Run with representative data (generates .gcda files)
./app_instr
# Creates: workload.gcda (profile data)

# Step 3: Optimized build using profile
g++ -O2 -std=c++20 -fprofile-use -o app_pgo workload.cpp

# === Clang PGO (IR-based, more accurate) ===
# Step 1: Instrument
clang++ -O2 -std=c++20 -fprofile-instr-generate -o app_instr workload.cpp

# Step 2: Profile
./app_instr
# Creates: default.profraw
llvm-profdata merge default.profraw -o workload.profdata

# Step 3: Optimize
clang++ -O2 -std=c++20 -fprofile-instr-use=workload.profdata -o app_pgo workload.cpp

# === CMake integration ===
# cmake -DCMAKE_CXX_FLAGS="-fprofile-generate" -B build-instr
# cmake --build build-instr && ./build-instr/app
# cmake -DCMAKE_CXX_FLAGS="-fprofile-use" -B build-pgo
# cmake --build build-pgo

```

### Q2: Measure PGO performance improvement

```bash

# Compare execution times:
$ time ./app_nopgo     # regular -O2 build
# real 0m1.82s

$ time ./app_pgo       # PGO-optimized build
# real 0m1.45s         # ~20% faster

# Why PGO helps here:
# 1. classify(): compiler reorders branches so case 3 (50%) is checked first
# 2. switch: compiler uses computed goto for hot cases
# 3. process(): hot loop gets better register allocation
# 4. code layout: hot path stays in instruction cache

# Verify with perf stat:
$ perf stat ./app_nopgo
#  5,200,000  branch-misses  (4.2% of branches)

$ perf stat ./app_pgo
#  1,100,000  branch-misses  (0.9% of branches)  ← 4x fewer

```

### Q3: Risks of stale profiles

```cpp

Stale profile problem:

v1.0 code:                  v2.0 code (changed control flow):
  if (x < 100) {              if (is_premium(x)) {    // NEW
    hot_path();   <-- 90%       premium_path();        // never profiled!
  }                           }
  Profile says: hot_path=90%  Profile: hot_path=90%... but it's gone!
  Compiler trusts this.       Compiler misoptimizes based on stale data.

```

When to regenerate profiles:

- **After any control-flow changes** (new branches, removed functions)
- **After changing hot data structures** (different memory access patterns)
- **Before every release build** (best practice)
- **NOT needed for:** comment changes, formatting, adding new cold paths

```bash

# Clang: detect stale profiles
clang++ -fprofile-instr-use=old.profdata \
        -Wprofile-instr-out-of-date \
        workload_v2.cpp
# warning: profile data may be out of date: of 10 functions,
#          3 have mismatched data that will be ignored

# GCC: similar warning
g++ -fprofile-use -Wmissing-profile workload_v2.cpp
# warning: profile count data file not found for function 'is_premium'

# CI best practice: automate PGO in pipeline
# 1. Build instrumented binary
# 2. Run benchmark suite to collect profile
# 3. Build optimized binary with fresh profile
# 4. Run tests on optimized binary
# 5. Deploy

```

---

## Notes

- PGO typically gives 10-30% speedup for branch-heavy code.
- Profile data is compiler-specific (GCC profiles don't work with Clang).
- Use representative workloads for profiling; benchmark suites work well.
- Clang's IR-based PGO (`-fprofile-instr-*`) is more accurate than frontend PGO.
- BOLT (Facebook's Binary Optimization and Layout Tool) is post-link PGO for even more gains.
