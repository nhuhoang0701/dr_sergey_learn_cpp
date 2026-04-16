# Use hardware performance counters with perf stat for cache and branch analysis

**Category:** Tooling & Debugging  
**Item:** #270  
**Reference:** <https://perf.wiki.kernel.org>  

---

## Topic Overview

Modern CPUs have **hardware performance counters** that track cache misses, branch mispredictions, instructions retired, and more. `perf stat` reads these counters with near-zero overhead.

| Counter | What It Measures | Bad Sign |
| --- | --- | --- |
| `cache-misses` | L1/LLC cache misses | High miss rate = bad data locality |
| `branch-misses` | Mispredicted branches | High rate = unpredictable control flow |
| `instructions` | Total instructions retired | Compare with cycles for IPC |
| `cycles` | CPU clock cycles | High cycles/low instructions = stalls |
| `L1-dcache-load-misses` | L1 data cache misses | Memory-bound workload |

---

## Self-Assessment

### Q1: Run `perf stat` and interpret the output

```cpp

// cache_test.cpp — compile: g++ -O2 -std=c++20 cache_test.cpp -o cache_test
#include <vector>
#include <numeric>
#include <iostream>
#include <random>

int main() {
    constexpr int N = 10'000'000;
    std::vector<int> data(N);
    std::iota(data.begin(), data.end(), 0);

    // Sequential access (cache-friendly)
    long long sum = 0;
    for (int i = 0; i < N; ++i)
        sum += data[i];

    std::cout << "sum = " << sum << '\n';
}

```

```bash

# Basic perf stat:
$ perf stat ./cache_test
#  Performance counter stats for './cache_test':
#       45.23 msec  task-clock
#  142,000,000      cycles
#   85,000,000      instructions     # 0.60 insn per cycle
#       12,345      cache-misses     # 0.02% of cache refs
#          234      branch-misses    # 0.01% of branches

# Specific counters:
$ perf stat -e cache-misses,cache-references,branch-misses,branches,\
L1-dcache-load-misses,L1-dcache-loads ./cache_test

#  Performance counter stats:
#       12,345  cache-misses         # 0.02% of cache-references
#   54,321,000  cache-references
#          234  branch-misses        # 0.01% of branches
#   20,000,000  branches
#       45,678  L1-dcache-load-misses # 0.05% of L1-dcache-loads
#  100,000,000  L1-dcache-loads

# Interpretation:
# - Very low cache miss rate (0.02%) = sequential access is cache-friendly
# - Very low branch miss rate (0.01%) = predictable loop pattern
# - IPC of 0.60 suggests memory latency isn't a bottleneck

```

### Q2: Data layout change that halves cache misses

```cpp

// sos_vs_aos.cpp
#include <vector>
#include <chrono>
#include <iostream>
#include <random>

// Array of Structures (AoS) — poor cache utilization
struct ParticleAoS {
    float x, y, z;          // position (used in update)
    float r, g, b, a;       // color (NOT used in update)
    float mass, charge;     // physics (NOT used in update)
    int id;                 // metadata
};

// Structure of Arrays (SoA) — cache-friendly
struct ParticlesSoA {
    std::vector<float> x, y, z;       // positions together
    std::vector<float> r, g, b, a;    // colors together
    std::vector<float> mass, charge;
    std::vector<int> id;
};

void update_positions_aos(std::vector<ParticleAoS>& particles, float dt) {
    for (auto& p : particles) {
        p.x += dt;  // Loads entire 40-byte struct into cache
        p.y += dt;  // even though we only need x, y, z (12 bytes)
        p.z += dt;  // ~70% of cache line is wasted!
    }
}

void update_positions_soa(ParticlesSoA& p, float dt) {
    const int n = p.x.size();
    for (int i = 0; i < n; ++i) p.x[i] += dt;  // Sequential float access
    for (int i = 0; i < n; ++i) p.y[i] += dt;  // 100% cache utilization
    for (int i = 0; i < n; ++i) p.z[i] += dt;
}

int main() {
    constexpr int N = 5'000'000;

    // AoS
    std::vector<ParticleAoS> aos(N);
    auto t1 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < 10; ++r)
        update_positions_aos(aos, 0.01f);
    auto t2 = std::chrono::high_resolution_clock::now();

    // SoA
    ParticlesSoA soa;
    soa.x.resize(N); soa.y.resize(N); soa.z.resize(N);
    auto t3 = std::chrono::high_resolution_clock::now();
    for (int r = 0; r < 10; ++r)
        update_positions_soa(soa, 0.01f);
    auto t4 = std::chrono::high_resolution_clock::now();

    auto ms_aos = std::chrono::duration<double, std::milli>(t2 - t1).count();
    auto ms_soa = std::chrono::duration<double, std::milli>(t4 - t3).count();
    std::cout << "AoS: " << ms_aos << " ms\n";
    std::cout << "SoA: " << ms_soa << " ms\n";
}

```

```bash

# Measure with perf:
$ perf stat -e L1-dcache-load-misses ./aos_benchmark
# AoS: ~850,000 L1 misses
# SoA: ~350,000 L1 misses  (~60% fewer!)

# Why: AoS loads 40 bytes per particle but only reads 12
# SoA stores x/y/z contiguously, so cache lines are fully utilized

```

### Q3: Use `perf record` + `perf report` to find cache-miss hotspots

```bash

# Record cache-miss events with call graphs:
$ perf record -e cache-misses -g ./cache_test
# [ perf record: Captured 12345 samples ]

# View interactive report:
$ perf report
#  Overhead  Command     Shared Object  Symbol
#  ........  ..........  ........
#   72.3%    cache_test  cache_test     update_positions_aos
#    15.1%   cache_test  libc.so        __memcpy
#     8.2%   cache_test  cache_test     main
#     4.4%   cache_test  [kernel]       

# Annotate source (needs -g debug info):
$ perf annotate update_positions_aos
#  Percent |  Source line
#  --------+----------------------------------------------
#   45.2%  |    p.x += dt;    <-- most cache misses here
#   22.1%  |    p.y += dt;
#   17.3%  |    p.z += dt;

# Record with specific event:
$ perf record -e L1-dcache-load-misses:u -g ./cache_test
# :u = user-space only (exclude kernel)

# Flamegraph from perf data:
$ perf script | stackcollapse-perf.pl | flamegraph.pl > cache_misses.svg

```

---

## Notes

- `perf stat` has near-zero overhead (reads HW counters, no sampling).
- `perf record` has overhead proportional to sampling frequency.
- Use `-e` to select specific events; `perf list` shows all available events.
- On WSL2/Docker, you may need `--privileged` or `perf_event_paranoid=0`.
- Intel VTune and AMD uProf provide richer analysis on their respective CPUs.
