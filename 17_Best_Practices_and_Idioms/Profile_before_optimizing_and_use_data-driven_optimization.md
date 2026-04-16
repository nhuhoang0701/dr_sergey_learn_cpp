# Profile before optimizing and use data-driven optimization

**Category:** Best Practices & Idioms  
**Item:** #137  
**Reference:** <https://en.cppreference.com/w/cpp/language/performance>  

---

## Topic Overview

*"Premature optimization is the root of all evil"* — Donald Knuth. **Always measure first, then optimize the measured hotspot.**

### The Optimization Workflow

```cpp

1. Write correct code first
2. Profile under realistic workload
3. Identify the hotspot (top 1-3 functions)
4. Hypothesize improvement
5. Implement change
6. Re-profile to verify improvement
7. If not faster, revert!

```

### Profiling Tools

| Tool | Platform | What it measures |
| --- | --- | --- |
| `perf stat/record` | Linux | CPU cycles, instructions, cache misses |
| Valgrind/Cachegrind | Linux | Cache misses, instruction count |
| VTune | Cross-platform | CPU microarchitecture, threading |
| `std::chrono` | Portable | Wall clock time (micro-benchmarks) |
| Google Benchmark | Portable | Statistical micro-benchmarking |
| Visual Studio Profiler | Windows | CPU sampling, memory |

---

## Self-Assessment

### Q1: Profile a program and identify the hottest function

```cpp

#include <chrono>
#include <cmath>
#include <iostream>
#include <numeric>
#include <vector>

// Simulate a program with multiple functions
void fast_function(std::vector<double>& data) {
    for (auto& x : data) x = x + 1.0;
}

void slow_function(std::vector<double>& data) {
    for (auto& x : data) x = std::sin(x) * std::cos(x) * std::exp(-std::abs(x) * 0.001);
}

void medium_function(std::vector<double>& data) {
    std::sort(data.begin(), data.end());
}

int main() {
    constexpr size_t N = 1'000'000;
    std::vector<double> data(N);
    std::iota(data.begin(), data.end(), 0.0);

    // Manual profiling with chrono
    auto measure = [](auto&& func, auto&&... args) {
        auto start = std::chrono::high_resolution_clock::now();
        func(std::forward<decltype(args)>(args)...);
        auto end = std::chrono::high_resolution_clock::now();
        return std::chrono::duration<double, std::milli>(end - start).count();
    };

    double t1 = measure(fast_function, data);
    double t2 = measure(slow_function, data);     // <-- likely hottest
    double t3 = measure(medium_function, data);

    std::cout << "fast_function:   " << t1 << " ms\n";
    std::cout << "slow_function:   " << t2 << " ms (HOTSPOT)\n";
    std::cout << "medium_function: " << t3 << " ms\n";
    std::cout << "Total: " << (t1 + t2 + t3) << " ms\n";
    std::cout << "Hotspot is " << (t2 / (t1 + t2 + t3) * 100) << "% of total\n";
}
// Typical output:
// fast_function:   ~1 ms
// slow_function:   ~50 ms (HOTSPOT)
// medium_function: ~70 ms
// Hotspot is ~40% of total
//
// Linux: perf record ./program && perf report
// Shows: slow_function at top of profile

```

### Q2: Show a premature optimization that hurt readability without measurable benefit

```cpp

#include <chrono>
#include <iostream>
#include <vector>

// "Optimized" version: bit tricks, unreadable
int count_set_bits_clever(int n) {
    n = n - ((n >> 1) & 0x55555555);
    n = (n & 0x33333333) + ((n >> 2) & 0x33333333);
    return (((n + (n >> 4)) & 0x0F0F0F0F) * 0x01010101) >> 24;
}

// Simple version: readable
int count_set_bits_simple(int n) {
    int count = 0;
    while (n) { count += n & 1; n >>= 1; }
    return count;
}

// Modern C++20: use the standard library
// #include <bit>
// int count_set_bits_modern(int n) { return std::popcount(static_cast<unsigned>(n)); }

int main() {
    constexpr int N = 10'000'000;
    int sink = 0;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < N; ++i) sink += count_set_bits_clever(i);
    auto t2 = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < N; ++i) sink += count_set_bits_simple(i);
    auto t3 = std::chrono::high_resolution_clock::now();

    auto clever_ms = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto simple_ms = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << "Clever: " << clever_ms << " ms\n";
    std::cout << "Simple: " << simple_ms << " ms\n";
    std::cout << "(sink=" << sink << ")\n";
    // Modern compilers optimize the simple version equally well!
    // The clever version added complexity for no benefit.
}
// Typical output (-O2):
// Clever: ~25 ms
// Simple: ~25 ms   <-- same speed! Compiler optimizes both.
// Lesson: the compiler is smarter than you think.

```

### Q3: Apply a data-oriented refactoring to a hot loop and measure the speedup

```cpp

#include <chrono>
#include <cmath>
#include <iostream>
#include <vector>

// BEFORE: AoS with pointer chasing and virtual calls
struct ParticleSlow {
    double x, y, z;
    double vx, vy, vz;
    double mass;
    char padding[64];  // simulate real-world bloated struct
};

void update_slow(std::vector<ParticleSlow>& particles, double dt) {
    for (auto& p : particles) {
        p.x += p.vx * dt;  // accesses scattered through 128-byte structs
        p.y += p.vy * dt;
        p.z += p.vz * dt;
    }
}

// AFTER: SoA, hot data contiguous
struct ParticlesFast {
    std::vector<double> x, y, z;
    std::vector<double> vx, vy, vz;
    // mass, padding not in hot path
};

void update_fast(ParticlesFast& p, double dt, size_t n) {
    for (size_t i = 0; i < n; ++i) p.x[i] += p.vx[i] * dt;
    for (size_t i = 0; i < n; ++i) p.y[i] += p.vy[i] * dt;
    for (size_t i = 0; i < n; ++i) p.z[i] += p.vz[i] * dt;
}

int main() {
    constexpr size_t N = 2'000'000;
    constexpr double dt = 0.016;

    // Setup AoS
    std::vector<ParticleSlow> slow(N);
    for (size_t i = 0; i < N; ++i)
        slow[i] = {double(i), 0, 0, 1, 2, 3, 1, {}};

    // Setup SoA
    ParticlesFast fast;
    fast.x.resize(N); fast.y.resize(N); fast.z.resize(N);
    fast.vx.resize(N); fast.vy.resize(N); fast.vz.resize(N);
    for (size_t i = 0; i < N; ++i) {
        fast.x[i] = double(i); fast.vx[i] = 1;
        fast.vy[i] = 2; fast.vz[i] = 3;
    }

    auto t1 = std::chrono::high_resolution_clock::now();
    update_slow(slow, dt);
    auto t2 = std::chrono::high_resolution_clock::now();
    update_fast(fast, dt, N);
    auto t3 = std::chrono::high_resolution_clock::now();

    auto ms_slow = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto ms_fast = std::chrono::duration<double, std::milli>(t3 - t2).count();

    std::cout << "AoS (slow): " << ms_slow << " ms\n";
    std::cout << "SoA (fast): " << ms_fast << " ms\n";
    std::cout << "Speedup: " << (ms_slow / ms_fast) << "x\n";
}
// Typical output:
// AoS (slow): ~30 ms
// SoA (fast): ~8 ms
// Speedup: ~3.7x

```

---

## Notes

- **Amdahl's Law:** If a function is 5% of runtime, a 2x speedup saves only 2.5% total time. Optimize the 80%, not the 5%.
- Always compile with `-O2` or `-O3` for profiling. Debug builds have different hotspots.
- Use `[[gnu::noinline]]` or `__attribute__((noinline))` to prevent inlining when profiling specific functions.
- Google Benchmark provides statistical analysis: mean, median, stddev across runs.
