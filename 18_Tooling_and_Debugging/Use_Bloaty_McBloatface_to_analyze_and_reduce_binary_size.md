# Use Bloaty McBloatface to analyze and reduce binary size

**Category:** Tooling & Debugging  
**Item:** #515  
**Reference:** <https://github.com/google/bloaty>  

---

## Topic Overview

Bloaty McBloatface analyzes ELF/Mach-O/PE binaries to show what contributes to binary size. It can break down by: sections, symbols, compilation units, or source files.

```cpp

Install: sudo apt install bloaty  (or build from source)
Usage:   bloaty ./myapp
         bloaty -d compileunits ./myapp
         bloaty -d symbols ./myapp
         bloaty ./after -- ./before    (diff mode)

```

---

## Self-Assessment

### Q1: Run Bloaty and identify top binary size contributors

```bash

# Build a sample project
g++ -std=c++20 -O2 -g -o myapp main.cpp

# Default: show sections
bloaty ./myapp

```

Typical output:

```text

    FILE SIZE        VM SIZE
 --------------  --------------
  34.5%  1.20Mi  42.1%  1.20Mi    .text          (code)
  26.3%   940Ki   0.0%       0    .debug_info    (debug data)
  15.7%   560Ki   0.0%       0    .debug_str     (debug strings)
   8.2%   293Ki  10.3%   293Ki    .rodata        (read-only data)
   5.1%   182Ki   6.4%   182Ki    .eh_frame      (exception handling)
   3.8%   136Ki   4.8%   136Ki    .data          (initialized data)
   2.4%    86Ki   3.0%    86Ki    .bss           (uninitialized data)

```

```bash

# Drill down by symbols (find biggest functions)
bloaty -d symbols -n 10 ./myapp

```

```cpp

# Typical top offenders:
  12.3%  std::__cxx11::basic_string<...>::_M_mutate(...)    # string ops
  8.7%   std::vector<Widget>::_M_realloc_insert(...)        # vector growth
  6.2%   __cxa_throw                                        # exception support
  4.1%   MyApp::processData()                               # your code

```

### Q2: `-O2` vs `-Os` binary size and performance

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <numeric>
#include <chrono>

void process(std::vector<int>& data) {
    std::sort(data.begin(), data.end());
    auto sum = std::accumulate(data.begin(), data.end(), 0LL);
    std::cout << "Sum: " << sum << '\n';
}

int main() {
    std::vector<int> data(1'000'000);
    std::iota(data.begin(), data.end(), 0);

    auto start = std::chrono::steady_clock::now();
    process(data);
    auto end = std::chrono::steady_clock::now();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start);
    std::cout << "Time: " << ms.count() << " ms\n";
}

```

```bash

# Build with different flags and compare:
g++ -std=c++20 -O2 -o prog_O2 prog.cpp && strip prog_O2
g++ -std=c++20 -Os -o prog_Os prog.cpp && strip prog_Os

bloaty prog_Os -- prog_O2

```

Typical diff output:

```text

     FILE SIZE       VM SIZE
  --------------  --------------
  -18.2%  -42Ki  -18.2%  -42Ki    .text        # -Os: less code (no unrolling)
   +0.0%      0   +0.0%      0    .rodata      # usually same
  -12.3%  -42Ki  -12.3%  -42Ki    TOTAL

```

| Metric | `-O2` | `-Os` | Difference |
| --- | --- | --- | --- |
| Binary size | ~230KB | ~188KB | -18% |
| Runtime | ~45ms | ~52ms | +15% slower |
| Best for | Servers, desktop | Embedded, IoT | Depends on constraints |

### Q3: `-fvisibility=hidden` + LTO for maximum size reduction

```bash

# Baseline
g++ -std=c++20 -O2 -o baseline prog.cpp && strip baseline

# With hidden visibility (only exported symbols visible)
g++ -std=c++20 -O2 -fvisibility=hidden -o hidden prog.cpp && strip hidden

# With LTO (link-time optimization)
g++ -std=c++20 -O2 -flto -o lto prog.cpp && strip lto

# Combined: hidden + LTO
g++ -std=c++20 -O2 -fvisibility=hidden -flto -o combined prog.cpp && strip combined

# Compare all against baseline:
bloaty combined -- baseline

```

| Technique | Size Reduction | How It Works |
| --- | --- | --- |
| `-fvisibility=hidden` | 5-15% | Hides internal symbols; linker can remove unused |
| `-flto` | 10-20% | Cross-TU inlining + dead code elimination |
| Combined | 15-30% | Synergy: LTO + fewer visible symbols = more can be removed |
| + `-Os` | 25-40% | Stack all three for maximum reduction |

```cmake

# CMakeLists.txt for maximum size reduction:
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)  # LTO
set(CMAKE_CXX_VISIBILITY_PRESET hidden)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
add_compile_options(-Os)

```

---

## Notes

- Bloaty can diff two binaries: `bloaty new -- old` shows what changed.
- Use `-d compileunits` to find which `.cpp` files generate the most code.
- Template-heavy code (STL, Boost) often dominates binary size.
- Debug info (`-g`) bloats file size but not VM size (not loaded at runtime).
- `strip` removes debug symbols; use `objcopy --only-keep-debug` to save them separately.
