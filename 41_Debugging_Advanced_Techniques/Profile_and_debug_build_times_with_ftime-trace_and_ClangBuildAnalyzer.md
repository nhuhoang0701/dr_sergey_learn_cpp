# Profile and debug build times with -ftime-trace and ClangBuildAnalyzer

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://aras-p.info/blog/2019/01/16/time-trace-timeline-flame-chart-profiler-for-Clang/>  

---

## Topic Overview

Slow builds waste millions of developer hours. If you've ever waited 30 seconds to recompile a single file, or watched a full clean build crawl along for minutes, you probably just accepted it as the cost of C++. You don't have to. Clang's `-ftime-trace` flag generates a detailed flame chart showing exactly where compilation time is being spent - header parsing, template instantiation, code generation, and optimization - so you can fix the actual bottleneck instead of guessing.

### Generating Time Traces

The workflow is straightforward: compile with the extra flag, and Clang writes a `.json` trace file alongside the object file. Then you load that file in Chrome's tracing viewer to see the timeline:

```bash
# Compile with -ftime-trace
clang++ -ftime-trace -std=c++20 -O2 main.cpp -o main

# This creates main.json alongside main.o
# Open in Chrome: chrome://tracing/ -> Load -> main.json
# Or use: https://ui.perfetto.dev/

# Shows:
# - Header parse times (which #includes are expensive)
# - Template instantiation times (which templates are slow)
# - Code generation times
# - Optimization pass times
```

Each bar in the flame chart corresponds to one compilation activity. Wide bars are the expensive ones. You'll often find that a single `#include` - frequently `<iostream>` or a complex Boost header - accounts for a disproportionate share of the time.

### ClangBuildAnalyzer (Aggregate Analysis)

Per-file traces are useful, but a real project has hundreds of translation units. ClangBuildAnalyzer aggregates all of those traces into a single report that ranks the worst offenders across the whole codebase. It tells you which header is the most expensive to parse (summed across every file that includes it) and which template instantiation is costing you the most time overall:

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

When you see `std::vector<std::string>::push_back` taking 450ms summed across all files, that's a signal that many translation units are independently instantiating the same template. That's exactly what `extern template` is designed to fix.

### CMake Integration

If you're using CMake, you can turn on `-ftime-trace` for all targets in one place using a generator expression so it only applies when compiling with Clang:

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

This way the traces are generated automatically on every build without needing to remember the flag each time.

### Common Build Time Fixes

Once you know what's slow, here's how the findings typically map to fixes. The table is ordered roughly from most common to least common:

```cpp
Finding:                        Fix:
───────────────────────────────────────────────────────
<iostream> takes 200ms          Use <cstdio> or forward-declare
Heavy template instantiation    Extern template, explicit instantiation
Many includes per TU            Precompiled headers (PCH)
Large header-only libraries     Use compiled version
include-what-you-use violations Run iwyu and remove unnecessary includes
```

The `<iostream>` finding surprises people the first time they see it. Including `<iostream>` drags in a large portion of the standard streams infrastructure. If you only need `printf`, use `<cstdio>` instead. If you only need to format numbers, there are lighter options.

---

## Self-Assessment

### Q1: What does a -ftime-trace flame chart show

The X axis is time (milliseconds). The Y axis shows nested activities. Top-level bars are compilation phases (Frontend, Backend). Nested bars show: header parsing (each `#include`), template instantiation (each unique instantiation), code generation, and optimization passes. Wide bars are bottlenecks. A healthy build has many roughly equal-width bars; a problematic build has one or two dominant bars that dwarf everything else.

### Q2: How to find and fix the slowest header include

The process is to measure first, then act. Many people guess that a particular header is slow when a different one is actually the culprit:

```bash
# 1. Build with -ftime-trace
# 2. Open in chrome://tracing
# 3. Look for the widest "Source" bars under "Frontend"
# 4. That header is the bottleneck

# Fix: forward declaration header, precompiled header, or
# include-what-you-use to remove unnecessary includes
```

The `include-what-you-use` tool is particularly effective because it will find headers your code doesn't actually need - they were included transitively and stuck around.

### Q3: What are extern template declarations for build speed

The idea is to stop every translation unit from independently instantiating the same template. You declare the template `extern` in the header (telling each TU "don't instantiate this yourself - it exists elsewhere"), and provide one explicit instantiation in a single `.cpp` file:

```cpp
// widget.hpp - declare extern template
extern template class std::vector<Widget>;
// Prevents every TU from instantiating vector<Widget>

// widget.cpp - explicit instantiation (only once)
template class std::vector<Widget>;
```

The benefit compounds with project size. If 50 translation units all include `widget.hpp` and used to each instantiate `vector<Widget>`, you've replaced 50 instantiations with exactly one.

---

## Notes

- `-ftime-trace` is Clang-only. GCC has `-ftime-report` (less detailed).
- `include-what-you-use` (iwyu) finds unnecessary includes automatically.
- Precompiled headers can save 30-50% of build time for large projects.
- C++20 modules will eventually replace most header optimization techniques.
