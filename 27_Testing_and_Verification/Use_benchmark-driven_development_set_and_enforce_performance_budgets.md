# Use benchmark-driven development: set and enforce performance budgets

**Category:** Testing & Verification  
**Item:** #588  
**Reference:** <https://github.com/google/benchmark>  

---

## Topic Overview

This file focuses on **performance budgets in CI** — setting thresholds, comparing benchmark JSON outputs, and using `DoNotOptimize`/`ClobberMemory` correctly. (See files #686 and #766 for the core measure→optimize→measure workflow and micro-benchmark pitfalls.)

### Performance Budget Pipeline

```bash

git push → CI builds → benchmarks run → JSON output
                                            │
         ┌──────────────────────────────────┘
         ▼
  Compare against baseline JSON (from main branch)
         │
    ┌────┴────┐
    │ >5%     │ ≤5%
    │ slower? │ change?
    ├─────────┤
    │  FAIL   │  PASS
    │  Block  │  Merge
    │  PR     │  PR
    └─────────┘

```

### DoNotOptimize vs ClobberMemory

| Mechanism | What It Does | When to Use |
| --- | --- | --- |
| `DoNotOptimize(x)` | Treats `x` as if read by external code | On function return values |
| `ClobberMemory()` | Memory barrier — all writes flushed | After writing to containers/arrays |

---

## Self-Assessment

### Q1: Write a Google Benchmark that measures throughput of two competing string split implementations

**Answer:**

```cpp

#include <benchmark/benchmark.h>
#include <string>
#include <vector>
#include <sstream>
#include <cstring>

// ═══════════ Implementation 1: istringstream-based split ═══════════
std::vector<std::string> split_stream(const std::string& s, char delim) {
    std::vector<std::string> tokens;
    std::istringstream iss(s);
    std::string token;
    while (std::getline(iss, token, delim))
        tokens.push_back(std::move(token));
    return tokens;
}

// ═══════════ Implementation 2: find-based split (zero-copy candidate) ═══════════
std::vector<std::string> split_find(const std::string& s, char delim) {
    std::vector<std::string> tokens;
    size_t start = 0;
    size_t end;
    while ((end = s.find(delim, start)) != std::string::npos) {
        tokens.emplace_back(s, start, end - start);
        start = end + 1;
    }
    if (start < s.size())
        tokens.emplace_back(s, start);
    return tokens;
}

// ═══════════ Helper: generate test data ═══════════
std::string make_csv_line(int fields) {
    std::string line;
    for (int i = 0; i < fields; ++i) {
        if (i > 0) line += ',';
        line += "field_" + std::to_string(i);
    }
    return line;
}

// ═══════════ Benchmarks ═══════════
static void BM_SplitStream(benchmark::State& state) {
    auto line = make_csv_line(state.range(0));
    for (auto _ : state) {
        auto tokens = split_stream(line, ',');
        benchmark::DoNotOptimize(tokens.data());
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(int64_t(state.iterations()) * state.range(0));
}

static void BM_SplitFind(benchmark::State& state) {
    auto line = make_csv_line(state.range(0));
    for (auto _ : state) {
        auto tokens = split_find(line, ',');
        benchmark::DoNotOptimize(tokens.data());
        benchmark::ClobberMemory();
    }
    state.SetItemsProcessed(int64_t(state.iterations()) * state.range(0));
}

BENCHMARK(BM_SplitStream)->RangeMultiplier(10)->Range(10, 10000)
    ->Unit(benchmark::kMicrosecond);
BENCHMARK(BM_SplitFind)->RangeMultiplier(10)->Range(10, 10000)
    ->Unit(benchmark::kMicrosecond);

BENCHMARK_MAIN();

// Output:
// BM_SplitStream/10       0.3 us    BM_SplitFind/10       0.1 us
// BM_SplitStream/100      3.5 us    BM_SplitFind/100      0.8 us
// BM_SplitStream/1000     42  us    BM_SplitFind/1000     8.5 us
// BM_SplitStream/10000    520 us    BM_SplitFind/10000    95  us
// → split_find is ~5× faster (avoids stringstream overhead)

```

### Q2: Use benchmark::DoNotOptimize and benchmark::ClobberMemory correctly to prevent dead-code elimination

**Answer:**

```cpp

#include <benchmark/benchmark.h>
#include <vector>
#include <numeric>

// ═══════════ WRONG: compiler eliminates the work ═══════════
static void BM_Wrong_DeadCode(benchmark::State& state) {
    std::vector<int> data(1000);
    std::iota(data.begin(), data.end(), 0);

    for (auto _ : state) {
        int sum = 0;
        for (int x : data) sum += x;
        // BUG: 'sum' is never used → compiler removes the loop!
        // Result: 0 ns/op (lies!)
    }
}
BENCHMARK(BM_Wrong_DeadCode);

// ═══════════ CORRECT: DoNotOptimize on the result ═══════════
static void BM_Correct_DoNotOptimize(benchmark::State& state) {
    std::vector<int> data(1000);
    std::iota(data.begin(), data.end(), 0);

    for (auto _ : state) {
        int sum = 0;
        for (int x : data) sum += x;
        benchmark::DoNotOptimize(sum);  // ← Compiler must compute sum
    }
}
BENCHMARK(BM_Correct_DoNotOptimize);

// ═══════════ ClobberMemory: when modifying memory ═══════════
static void BM_Correct_ClobberMemory(benchmark::State& state) {
    std::vector<int> data(1000);

    for (auto _ : state) {
        // Modify the vector in place
        for (int i = 0; i < 1000; ++i)
            data[i] = i * i;

        benchmark::ClobberMemory();
        // ← Forces stores to be visible, prevents compiler
        //   from noticing that data isn't read afterward
        //   and eliminating the writes
    }
}
BENCHMARK(BM_Correct_ClobberMemory);

// ═══════════ Combined: both needed ═══════════
static void BM_Combined(benchmark::State& state) {
    std::vector<int> src(10000, 42);
    std::vector<int> dst(10000);

    for (auto _ : state) {
        std::copy(src.begin(), src.end(), dst.begin());
        benchmark::ClobberMemory();              // Force writes to dst
        benchmark::DoNotOptimize(dst.data());    // Pretend dst is read
    }
    state.SetBytesProcessed(int64_t(state.iterations()) * 10000 * sizeof(int));
}
BENCHMARK(BM_Combined);

BENCHMARK_MAIN();

```

### Q3: Set performance regression thresholds in CI by comparing JSON benchmark outputs between commits

**Answer:**

```bash

#!/bin/bash
# ═══════════ CI performance regression detection ═══════════

# Step 1: Run benchmarks on the BASELINE (main branch)
git checkout main
cmake --build build --target my_benchmarks
./build/my_benchmarks \
    --benchmark_out=baseline.json \
    --benchmark_out_format=json \
    --benchmark_repetitions=5 \
    --benchmark_report_aggregates_only=true

# Step 2: Run benchmarks on the PR branch
git checkout feature-branch
cmake --build build --target my_benchmarks
./build/my_benchmarks \
    --benchmark_out=current.json \
    --benchmark_out_format=json \
    --benchmark_repetitions=5 \
    --benchmark_report_aggregates_only=true

# Step 3: Compare with Google Benchmark's compare tool
# Outputs: benchmark name, old time, new time, % change
python3 tools/compare.py benchmarks baseline.json current.json

# Step 4: Fail CI if any benchmark regressed >5%
python3 -c "
import json, sys

with open('baseline.json') as f: base = json.load(f)
with open('current.json') as f:  curr = json.load(f)

base_times = {b['name']: b['cpu_time'] for b in base['benchmarks']
              if b.get('aggregate_name') == 'mean'}
curr_times = {b['name']: b['cpu_time'] for b in curr['benchmarks']
              if b.get('aggregate_name') == 'mean'}

threshold = 0.05  # 5% regression threshold
failed = False

for name, base_time in base_times.items():
    if name in curr_times:
        curr_time = curr_times[name]
        change = (curr_time - base_time) / base_time
        status = 'PASS' if change <= threshold else 'FAIL'
        if change > threshold:
            failed = True
        print(f'{status}: {name}: {base_time:.1f} -> {curr_time:.1f} ({change:+.1%})')

sys.exit(1 if failed else 0)
"

```

```yaml

# ═══════════ GitHub Actions CI integration ═══════════
# .github/workflows/perf.yml
name: Performance Check
on: [pull_request]

jobs:
  benchmark:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

        with:
          fetch-depth: 0  # Need full history for baseline

      - name: Build benchmarks

        run: |
          cmake -B build -DCMAKE_BUILD_TYPE=Release
          cmake --build build --target my_benchmarks

      - name: Run current benchmarks

        run: ./build/my_benchmarks --benchmark_out=current.json --benchmark_out_format=json

      - name: Get baseline

        run: |
          git stash
          git checkout main
          cmake --build build --target my_benchmarks
          ./build/my_benchmarks --benchmark_out=baseline.json --benchmark_out_format=json
          git checkout -

      - name: Compare

        run: python3 scripts/check_perf_budget.py baseline.json current.json --threshold 5

```

---

## Notes

- `--benchmark_repetitions=5` → runs each benchmark 5 times, reports mean/median/stddev
- `--benchmark_min_time=2.0` → ensures enough samples for statistical significance
- Store `baseline.json` as a CI artifact to avoid rebuilding main every time
- Consider `benchmark::Counter` for custom metrics (cache misses, allocations)
- For flaky benchmarks, use the `--benchmark_min_warmup_time` flag to stabilize results
