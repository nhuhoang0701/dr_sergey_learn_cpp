# Use compile-time benchmarking to track build performance regressions

**Category:** Tooling & Debugging  
**Item:** #512  
**Reference:** <https://github.com/aras-p/ClangBuildAnalyzer>  

---

## Topic Overview

Compile-time benchmarking detects when a code change makes builds slower. Tools like ClangBuildAnalyzer and `-ftime-trace` identify exactly which headers and templates are slow.

```cpp

CI pipeline:
  Build with -ftime-trace
       │
  ClangBuildAnalyzer
       │
  Report: "<regex> takes 180ms per TU"
          "Widget<T> instantiated 47 times"
       │
  Compare against baseline
       │
  Alert if regression > 10%

```

---

## Self-Assessment

### Q1: Find the slowest headers and templates with ClangBuildAnalyzer

```bash

# Step 1: Build with -ftime-trace
cmake -B build -DCMAKE_CXX_COMPILER=clang++ \
      -DCMAKE_CXX_FLAGS="-ftime-trace"
cmake --build build

# Step 2: Capture all trace files
ClangBuildAnalyzer --all build capture.bin

# Step 3: Analyze
ClangBuildAnalyzer --analyze capture.bin

```

Typical output:

```text

**** Files that took longest to compile (wall time):
  180ms  src/parser.cpp
  152ms  src/engine.cpp
   98ms  src/renderer.cpp

**** Templates that took longest to instantiate:
  45ms  std::vector<Widget> (18 times)
  32ms  std::map<std::string, Config> (12 times)
  28ms  fmt::format<...> (47 times)

**** Files that were included the most:
  47   <vector>
  42   <string>
  38   <iostream>     <-- every include costs ~15ms
  12   <regex>        <-- 180ms per include!

**** Expensive headers (time * include count):
  2160ms  <regex>       (12 includes * 180ms)
   810ms  <iostream>    (38 includes * ~21ms)
   450ms  <map>         (15 includes * ~30ms)

```

### Q2: Measure the cost of heavy headers

```bash

# Create minimal test files:
echo '#include <regex>' > test_regex.cpp
echo 'int main() {}' >> test_regex.cpp

echo '#include <iostream>' > test_iostream.cpp
echo 'int main() {}' >> test_iostream.cpp

echo 'int main() {}' > test_empty.cpp

# Measure with time:
time clang++ -std=c++20 -c test_empty.cpp     # ~0.05s baseline
time clang++ -std=c++20 -c test_iostream.cpp   # ~0.15s (+0.10s)
time clang++ -std=c++20 -c test_regex.cpp      # ~0.35s (+0.30s)

```

| Header | Approx. Compile Overhead | Lines Pulled In |
| --- | --- | --- |
| `<array>` | ~5ms | ~2K |
| `<vector>` | ~15ms | ~10K |
| `<string>` | ~20ms | ~12K |
| `<iostream>` | ~100ms | ~25K |
| `<map>` | ~30ms | ~15K |
| `<algorithm>` | ~25ms | ~14K |
| `<regex>` | ~300ms | ~50K |
| `<variant>` | ~40ms | ~8K |

```cpp

// BAD: includes <regex> in a widely-included header
// config.h
#include <regex>  // 300ms * N translation units = slow!

bool validate(const std::string& s) {
    return std::regex_match(s, std::regex("[a-z]+"));
}

// GOOD: move to .cpp file, keep header light
// config.h
#include <string>
bool validate(const std::string& s);  // forward declare only

// config.cpp
#include "config.h"
#include <regex>  // heavy include only in ONE TU
bool validate(const std::string& s) {
    return std::regex_match(s, std::regex("[a-z]+"));
}

```

### Q3: Forward declarations and include reduction for 20%+ compile time savings

Before:

```cpp

// widget.h (included by 30 files)
#include <vector>     // 15ms
#include <map>        // 30ms
#include <string>     // 20ms
#include <algorithm>  // 25ms
#include <iostream>   // 100ms = 190ms per includer!

class Widget {
    std::vector<int> data_;
    std::map<std::string, int> props_;
public:
    void process();
    void print() const;
};

```

After:

```cpp

// widget.h (optimized)
#include <vector>  // needed: data_ member
#include <string>  // needed: map key type
#include <map>     // needed: props_ member
// Removed <algorithm> and <iostream>: only needed in .cpp!

class Widget {
    std::vector<int> data_;
    std::map<std::string, int> props_;
public:
    void process();
    void print() const;
};

// widget.cpp
#include "widget.h"
#include <algorithm>  // moved here
#include <iostream>   // moved here

void Widget::process() { std::sort(data_.begin(), data_.end()); }
void Widget::print() const { for (auto& d : data_) std::cout << d << ' '; }

```

| Metric | Before | After | Savings |
| --- | --- | --- | --- |
| Header cost per TU | 190ms | 65ms | -66% |
| Total (30 TUs) | 5.7s | 1.95s | -66% |

---

## Notes

- Use `include-what-you-use` (IWYU) to automate include reduction.
- PCH helps when many TUs include the same heavy headers.
- Track compile times in CI: store build times per commit.
- Forward declare classes used only by pointer/reference in headers.
- `-ftime-trace-granularity=100` shows events as small as 100µs.
