# Write branchless code using conditional moves and arithmetic tricks

**Category:** Performance & CPU Architecture  
**Item:** #540  
**Reference:** <https://godbolt.org>  

---

## Topic Overview

Branches (if/else) cause pipeline stalls when mispredicted (~15-20 cycles penalty). Branchless code uses conditional moves (cmov) or arithmetic to avoid branches entirely.

```asm

Branch version:                     Branchless (cmov):
  cmp eax, ebx                        cmp eax, ebx
  jl .take_a                          cmovl ecx, eax    ; select a if a<b
  mov ecx, ebx                        cmovge ecx, ebx   ; select b otherwise
  jmp .done                           ; NO jump, NO prediction needed!
  .take_a:
  mov ecx, eax
  .done:
  // 15-20 cycle penalty if mispredicted  // Always ~2 cycles

```

| Pattern | Branch cost | Branchless cost | Use branchless? |
| --- | --- | --- | --- |
| Predictable (95%+ one way) | ~0 cycles | ~2 cycles | No |
| Unpredictable (50/50) | ~10 cycles avg | ~2 cycles | Yes |
| Both paths expensive | ~10 + path_cost | both_costs | No |

---

## Self-Assessment

### Q1: Branchless min and verify on Compiler Explorer

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>

// Branch version: if/else
int min_branch(int a, int b) {
    if (a < b) return a;  // cmp + jl (branch)
    return b;
    // Assembly:
    //   cmp edi, esi
    //   jl .L2          <- BRANCH (mispredicts ~50% for random data)
    //   mov eax, esi
    //   ret
    //   .L2: mov eax, edi
    //   ret
}

// Branchless version 1: ternary (compiler often uses cmov)
int min_ternary(int a, int b) {
    return a < b ? a : b;
    // Assembly (with -O2):
    //   cmp edi, esi
    //   cmovle eax, edi  <- CONDITIONAL MOVE (no branch!)
    //   ret
}

// Branchless version 2: arithmetic trick
int min_arith(int a, int b) {
    // x86 trick: uses the sign bit of (a - b)
    int diff = a - b;
    return b + (diff & (diff >> 31));
    // If a < b: diff < 0, diff>>31 = -1 (all 1s), diff & -1 = diff, b + diff = a
    // If a >= b: diff >= 0, diff>>31 = 0, diff & 0 = 0, b + 0 = b
    // WARNING: overflow if a-b overflows int range!
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> a(N), b(N);
    std::mt19937 rng(42);
    for (int i = 0; i < N; ++i) {
        a[i] = rng() % 1000;  // random: branch predictor fails ~50%
        b[i] = rng() % 1000;
    }

    auto bench = [&](auto fn, const char* label) {
        volatile int result = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < N; ++i) result = fn(a[i], b[i]);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench(min_branch,  "Branch  ");
    bench(min_ternary, "Ternary ");
    bench(min_arith,   "Arith   ");
    // Random data: Branch ~350ms, Ternary ~150ms, Arith ~160ms
    // Sorted data: Branch ~120ms (predictor succeeds), Ternary ~150ms
}
// Verify on godbolt.org: compile with -O2, look for cmovl (branchless)

```

### Q2: `std::min` compiles to cmov

```cpp

#include <iostream>
#include <algorithm>  // std::min
#include <vector>
#include <chrono>
#include <random>

// std::min is typically branchless at -O2!
int use_std_min(int a, int b) {
    return std::min(a, b);
    // GCC -O2 assembly:
    //   cmp edi, esi
    //   mov eax, esi
    //   cmovle eax, edi    <- cmov! branchless!
    //   ret
    //
    // Clang -O2 may also use:
    //   cmp edi, esi
    //   cmovl esi, edi
    //   mov eax, esi
    //   ret
}

// std::max is also branchless:
int use_std_max(int a, int b) {
    return std::max(a, b);
    // cmp + cmovge -> branchless
}

// std::clamp is two cmovs:
int use_std_clamp(int val, int lo, int hi) {
    return std::clamp(val, lo, hi);
    // cmp + cmovl + cmp + cmovg -> two conditional moves
}

// Absolute value: also branchless at -O2
int abs_branchless(int x) {
    return x < 0 ? -x : x;
    // Assembly:
    //   mov eax, edi
    //   neg eax
    //   cmovs eax, edi  <- conditional move on sign flag
    //   ret
    // Alternative: (x ^ (x >> 31)) - (x >> 31)  [two's complement trick]
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> data(N);
    std::mt19937 rng(42);
    for (auto& x : data) x = rng() % 2000 - 1000;

    volatile int r = 0;
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int i = 1; i < N; ++i)
        r = std::min(static_cast<int>(r), data[i]);
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << "std::min chain: " << ms.count() << " ms\n";  // ~120ms
    std::cout << "Result: " << r << '\n';  // minimum element
}

```

### Q3: When branchless code is slower

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <string>

// Case 1: PREDICTABLE branches are FASTER than branchless
bool mostly_true(int x) { return x > 0; }  // 99% true

void branch_predictable(const int* data, int n) {
    volatile int sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] > 0)  // 99% taken -> predictor almost perfect
            sum += data[i];
        // Branch: ~0 cycles (99% predicted correctly)
        // cmov: ~2 cycles EVERY time -> worse!
    }
}

// Case 2: EXPENSIVE side effects -- can't use branchless
void expensive_paths(const std::vector<int>& keys,
                     const std::vector<std::string>& cache) {
    for (int k : keys) {
        if (k < static_cast<int>(cache.size())) {
            // Cheap path: read from cache
            volatile auto& s = cache[k];
        } else {
            // Expensive path: would need DB lookup, allocation, etc.
            // Branchless would execute BOTH paths -> wasteful!
        }
    }
}

// Case 3: Floating-point -- no cmov (uses SSE min/max instead)
float fmin_branch(float a, float b) {
    return a < b ? a : b;
    // Uses: vminss xmm0, xmm0, xmm1 (SSE min, always branchless)
    // Compiler ALWAYS makes this branchless for float!
}

int main() {
    constexpr int N = 100'000'000;
    std::vector<int> mostly_pos(N);
    std::mt19937 rng(42);
    // 99% positive values -> branch is predictable
    for (auto& x : mostly_pos)
        x = (rng() % 100 < 99) ? (rng() % 1000 + 1) : -(rng() % 1000);

    std::cout << "When NOT to use branchless code:\n";
    std::cout << "  1. Highly predictable branch (>95% one direction)\n";
    std::cout << "     -> predictor costs ~0, cmov costs ~2 cycles\n";
    std::cout << "  2. Paths with side effects or expensive work\n";
    std::cout << "     -> cmov evaluates BOTH sides always\n";
    std::cout << "  3. Floating-point comparisons\n";
    std::cout << "     -> compiler already uses minss/maxss (branchless)\n";
    std::cout << "\nProfile first: perf stat -e branch-misses ./app\n";
    std::cout << "If branch-miss rate < 1%: branch is fine, don't optimize\n";
}

```

---

## Notes

- `std::min`, `std::max`, `std::clamp` all compile to cmov at `-O2`.
- Use Compiler Explorer to verify: look for `cmov` in the assembly.
- Branchless wins when branch misprediction rate > 5-10%.
- `perf stat -e branch-misses,branches` tells you the misprediction rate.
- For SIMD: all comparisons are inherently branchless (mask + blend).
