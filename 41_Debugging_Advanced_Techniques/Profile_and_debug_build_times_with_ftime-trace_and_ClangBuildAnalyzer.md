# Profile and debug build times with -ftime-trace and ClangBuildAnalyzer

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://aras-p.info/blog/2019/01/16/time-trace-timeline-flame-chart-profiler-for-Clang/>  

---

## Topic Overview

Slow builds waste millions of developer hours. Clang's `-ftime-trace` generates flame charts showing exactly where compilation time is spent.

### Generating Time Traces

```bash

# Compile with -ftime-trace
clang++ -ftime-trace -std=c++20 -O2 main.cpp -o main

# This creates main.json alongside main.o
# Open in Chrome: chrome://tracing/ → Load → main.json
# Or use: https://ui.perfetto.dev/

# Shows:
# - Header parse times (which #includes are expensive)
# - Template instantiation times (which templates are slow)
# - Code generation times
# - Optimization pass times

```

### ClangBuildAnalyzer (Aggregate Analysis)

```bash

# Install
git clone https://github.com/ArasP/ClangBuildAnalyzer && cd ClangBuildAnalyzer
cmake -B build && cmake --build build

# Capture all time traces from a full build:
ClangBuildAnalyzer --start build/
cmake --build build/     # Normal build with -ftime-trace
ClangBuildAnalyzer --stop build/ capture.bin

# Analyze:
ClangBuildAnalyzer --analyze capture.bin

# Output shows:
# Top 10 slowest headers to parse:
#   236ms  /usr/include/c++/12/iostream
#   189ms  /usr/include/c++/12/vector
#
# Top 10 slowest template instantiations:
#   450ms  std::vector<std::string>::push_back
#   380ms  std::map<std::string, std::vector<int>>::insert

```

### CMake Integration

```cmake

# Add -ftime-trace to all targets
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
    add_compile_options(-ftime-trace)
endif()

# Or per-target:
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:Clang>:-ftime-trace>
)

```

### Common Build Time Fixes

```cpp

Finding:                        Fix:
───────────────────────────────────────────────────────
<iostream> takes 200ms          Use <cstdio> or forward-declare
Heavy template instantiation    Extern template, explicit instantiation
Many includes per TU            Precompiled headers (PCH)
Large header-only libraries     Use compiled version
include-what-you-use violations Run iwyu and remove unnecessary includes

```

---

## Self-Assessment

### Q1: What does a -ftime-trace flame chart show

The X axis is time (milliseconds). The Y axis shows nested activities. Top-level bars are compilation phases (Frontend, Backend). Nested bars show: header parsing (each `#include`), template instantiation (each unique instantiation), code generation, and optimization passes. Wide bars are bottlenecks.

### Q2: How to find and fix the slowest header include

```bash

# 1. Build with -ftime-trace
# 2. Open in chrome://tracing
# 3. Look for the widest "Source" bars under "Frontend"
# 4. That header is the bottleneck

# Fix: forward declaration header, precompiled header, or
# include-what-you-use to remove unnecessary includes

```

### Q3: What are extern template declarations for build speed

```cpp

// widget.hpp — declare extern template
extern template class std::vector<Widget>;
// Prevents every TU from instantiating vector<Widget>

// widget.cpp — explicit instantiation (only once)
template class std::vector<Widget>;

```

---

## Notes

- `-ftime-trace` is Clang-only. GCC has `-ftime-report` (less detailed).
- `include-what-you-use` (iwyu) finds unnecessary includes automatically.
- Precompiled headers can save 30-50% of build time for large projects.
- C++20 modules will eventually replace most header optimization techniques.
