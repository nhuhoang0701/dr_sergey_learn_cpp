# Use perf flamegraphs for visual CPU profiling

**Category:** Tooling & Debugging  
**Item:** #804  
**Standard:** C++23  
**Reference:** <https://www.brendangregg.com/flamegraphs.html>  

---

## Topic Overview

Flamegraphs visualize where CPU time is spent. Each box represents a function; width = percentage of samples.

```cpp

Flamegraph (read bottom-up):
┌───────────────────────────────────────────────┐
│                 main()                          │
├────────────────────────┬──────────────────────┤
│    process_data() 60%    │   sort_results() 35% │
├──────────┬─────────────┤                      │
│ parse()  │ validate()   │                      │
│   40%    │    20%       │                      │
└──────────┴─────────────┴──────────────────────┘
  ^ widest frame = most time = optimize first

```

---

## Self-Assessment

### Q1: Generate a flamegraph with `perf record` + `flamegraph.pl`

```cpp

// hotspot.cpp — compile: g++ -O2 -g -std=c++20 hotspot.cpp -o hotspot
// -g is needed for symbol resolution in perf
#include <vector>
#include <algorithm>
#include <numeric>
#include <cmath>
#include <iostream>

void expensive_parse(std::vector<double>& data) {
    for (auto& d : data)
        d = std::sin(d) * std::cos(d) * std::log(d + 1.0);
}

void cheap_validate(const std::vector<double>& data) {
    for (const auto& d : data)
        if (std::isnan(d)) throw std::runtime_error("NaN!");
}

void sort_results(std::vector<double>& data) {
    std::sort(data.begin(), data.end());
}

int main() {
    std::vector<double> data(10'000'000);
    std::iota(data.begin(), data.end(), 1.0);

    expensive_parse(data);
    cheap_validate(data);
    sort_results(data);

    std::cout << "Done. First: " << data[0] << '\n';
}

```

```bash

# Step 1: Record CPU samples with call graph
$ perf record -g --call-graph dwarf -F 999 ./hotspot
# -F 999: sample frequency (999 Hz avoids aliasing with timer)
# -g --call-graph dwarf: accurate stack traces

# Step 2: Generate flamegraph
$ perf script | stackcollapse-perf.pl | flamegraph.pl > flamegraph.svg
# Tools from: https://github.com/brendangregg/FlameGraph

# Step 3: Open in browser
$ firefox flamegraph.svg  # or any browser
# Interactive: click to zoom into a subtree, search for function names

# Alternative: perf's built-in flamegraph (Linux 5.8+)
$ perf script report flamegraph

```

### Q2: Identify the hottest function and optimize it

Reading the flamegraph from Q1:

```cpp

Flame graph analysis:
─────────────────────────────────────────────
  main()                                  100%
  ├─ expensive_parse()                    55%  <-- WIDEST = optimize!
  │   ├─ sin()                            20%  (self-time)
  │   ├─ cos()                            18%
  │   └─ log()                            17%
  ├─ sort_results()                       40%
  │   └─ __introsort_loop()               38%  (self-time)
  └─ cheap_validate()                      5%  (ignore)

Optimization target: expensive_parse() at 55%

```

```cpp

// Optimized version: precompute sin*cos = 0.5*sin(2x)
void expensive_parse_opt(std::vector<double>& data) {
    for (auto& d : data)
        d = 0.5 * std::sin(2.0 * d) * std::log(d + 1.0);
    // Reduced from 3 trig calls to 1 (sin(2x) = 2*sin(x)*cos(x))
}

```

### Q3: Compare before/after flamegraphs

```bash

# Before optimization:
$ perf record -g --call-graph dwarf ./hotspot_before
$ perf script | stackcollapse-perf.pl > before.folded
$ flamegraph.pl before.folded > before.svg

# After optimization:
$ perf record -g --call-graph dwarf ./hotspot_after
$ perf script | stackcollapse-perf.pl > after.folded
$ flamegraph.pl after.folded > after.svg

# Differential flamegraph (red = slower, blue = faster):
$ difffolded.pl before.folded after.folded | flamegraph.pl > diff.svg

# Expected diff flamegraph:
# - expensive_parse: BLUE (reduced from 55% to 30%)
# - sort_results: wider (now proportionally larger, was 40%, now ~55%)
# - cheap_validate: unchanged

# Also compare with perf stat:
$ perf stat ./hotspot_before
# 2.45 seconds time elapsed
$ perf stat ./hotspot_after
# 1.78 seconds time elapsed  (27% faster!)

```

---

## Notes

- Always compile with `-g` (debug symbols) for meaningful function names.
- Use `-fno-omit-frame-pointer` for reliable stack traces with `--call-graph fp`.
- `--call-graph dwarf` is more accurate but slower than `--call-graph fp`.
- Differential flamegraphs are invaluable for validating optimizations.
- Online viewer: <https://www.speedscope.app/> imports `perf script` output.
- On macOS, use Instruments.app; on Windows, use ETW + WPA.
