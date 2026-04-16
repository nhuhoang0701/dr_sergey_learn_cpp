# Use Quick Bench or Google Benchmark for micro-benchmarking

**Category:** Tooling & Debugging  
**Item:** #148  
**Reference:** <https://github.com/google/benchmark>  

---

## Topic Overview

Google Benchmark provides a framework for writing accurate micro-benchmarks. Quick Bench (quick-bench.com) is an online front-end.

```text

Benchmark output:
-----------------------------------------------------------
Benchmark               Time             CPU   Iterations
-----------------------------------------------------------
BM_push_back          12.3 ns         12.2 ns    56000000
BM_emplace_back       10.1 ns         10.0 ns    68000000
                      ^^^^^^          ^^^^^^
                    wall clock       CPU time

```

---

## Self-Assessment

### Q1: Benchmark `push_back` vs `emplace_back`

```cpp

// bench.cpp — compile:
// g++ -O2 -std=c++20 bench.cpp -lbenchmark -lpthread -o bench
#include <benchmark/benchmark.h>
#include <vector>
#include <string>

static void BM_push_back(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(1000);
        for (int i = 0; i < 1000; ++i)
            v.push_back(std::string("hello world"));  // creates temp, then moves
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_push_back);

static void BM_emplace_back(benchmark::State& state) {
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(1000);
        for (int i = 0; i < 1000; ++i)
            v.emplace_back("hello world");  // constructs in-place, no temp
        benchmark::DoNotOptimize(v.data());
    }
}
BENCHMARK(BM_emplace_back);

// Parameterized benchmark:
static void BM_push_back_n(benchmark::State& state) {
    const int n = state.range(0);
    for (auto _ : state) {
        std::vector<std::string> v;
        v.reserve(n);
        for (int i = 0; i < n; ++i)
            v.push_back(std::string("hello"));
        benchmark::DoNotOptimize(v.data());
    }
    state.SetItemsProcessed(state.iterations() * n);
}
BENCHMARK(BM_push_back_n)->Range(8, 8 << 10);

BENCHMARK_MAIN();

```

```bash

# Build and run:
$ g++ -O2 -std=c++20 bench.cpp -lbenchmark -lpthread -o bench
$ ./bench
# ----------------------------------------------------------
# Benchmark              Time           CPU      Iterations
# ----------------------------------------------------------
# BM_push_back        15234 ns       15200 ns         46000
# BM_emplace_back     12456 ns       12400 ns         56000
#                     ~~~~~~ emplace_back ~18% faster
# BM_push_back_n/8       45 ns          45 ns      15000000
# BM_push_back_n/64     320 ns         319 ns       2100000
# BM_push_back_n/512   2890 ns        2880 ns        240000
# BM_push_back_n/4096 24500 ns       24400 ns         28000

```

### Q2: `DoNotOptimize` prevents dead code elimination

```cpp

#include <benchmark/benchmark.h>

// WITHOUT DoNotOptimize — compiler eliminates the work:
static void BM_bad(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1000; ++i)
            sum += i;          // Compiler sees sum is unused -> eliminates loop!
        // No DoNotOptimize -> measures NOTHING
    }
}
BENCHMARK(BM_bad);  // reports ~0 ns (useless!)

// WITH DoNotOptimize — compiler must keep the computation:
static void BM_good(benchmark::State& state) {
    for (auto _ : state) {
        int sum = 0;
        for (int i = 0; i < 1000; ++i)
            sum += i;
        benchmark::DoNotOptimize(sum);  // treats 'sum' as observable
    }
}
BENCHMARK(BM_good);  // reports actual loop time

// ClobberMemory forces memory writes to be visible:
static void BM_container(benchmark::State& state) {
    std::vector<int> v(1000);
    for (auto _ : state) {
        for (auto& x : v) x += 1;
        benchmark::ClobberMemory();  // writes to v are "observed"
    }
}
BENCHMARK(BM_container);

BENCHMARK_MAIN();

```

How `DoNotOptimize` works:

```cpp

// Simplified implementation:
template <class T>
void DoNotOptimize(T& value) {
    asm volatile("" : "+r"(value));  // value in register, compiler can't eliminate
}
// The asm volatile is a barrier: compiler must compute 'value' but the asm does nothing.

```

### Q3: Filter and interpret benchmark output

```bash

# Run only benchmarks matching a regex:
$ ./bench --benchmark_filter="emplace"
# Runs only BM_emplace_back

$ ./bench --benchmark_filter="BM_push_back_n/.*"
# Runs all parameterized variants

# Output columns explained:
# Benchmark       Time        CPU       Iterations
# BM_X           15.2 ns     15.1 ns    46000000
# |              |           |          |
# name           wall-clock  CPU-only   auto-determined
#                per iter    per iter   for statistical
#                                       stability

# Output formats:
$ ./bench --benchmark_format=json > results.json
$ ./bench --benchmark_format=csv  > results.csv

# Compare two runs:
$ ./bench --benchmark_out=baseline.json --benchmark_out_format=json
# ... make changes
$ ./bench --benchmark_out=improved.json --benchmark_out_format=json
$ python3 -m google_benchmark.compare baseline.json improved.json
# Benchmark              Time       CPU
# BM_push_back        -0.18%    -0.19%  (18% faster!)

# Statistical analysis (multiple runs):
$ ./bench --benchmark_repetitions=10 --benchmark_report_aggregates_only=true
# Shows mean, median, stddev

```

---

## Notes

- Quick Bench (quick-bench.com) runs Google Benchmark online — great for quick tests.
- Always benchmark with `-O2` or `-O3`; `-O0` results are meaningless.
- Use `benchmark::ClobberMemory()` for container modifications.
- `SetItemsProcessed()` and `SetBytesProcessed()` add throughput metrics.
- Install: `vcpkg install benchmark` or `apt install libbenchmark-dev`.
