# Use link-time optimization (LTO) to enable cross-TU inlining and devirtualization

**Category:** Tooling & Debugging  
**Item:** #412  
**Reference:** <https://llvm.org/docs/LinkTimeOptimization.html>  

---

## Topic Overview

LTO defers optimization to link time, enabling optimizations across translation unit (TU) boundaries:

```cpp

Without LTO:                          With LTO:
  a.cpp -> a.o  \                      a.cpp -> a.o (IR) \
  b.cpp -> b.o   > linker -> binary    b.cpp -> b.o (IR)  > optimizer -> binary
  c.cpp -> c.o  /                      c.cpp -> c.o (IR) /
  (each .o optimized                   (all IR merged,
   independently)                       whole-program optimization)

```

| Mode | Full LTO | Thin LTO |
| --- | --- | --- |
| Flag | `-flto` | `-flto=thin` |
| Parallelism | Single-threaded link | Parallel link |
| Build time | Slow (large projects) | Much faster |
| Optimization | Best possible | Nearly as good |
| Memory | High (all IR in memory) | Low (per-module) |

---

## Self-Assessment

### Q1: Enable LTO in CMake and measure impact

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(LTODemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)

# Check LTO support:
include(CheckIPOSupported)
check_ipo_supported(RESULT lto_supported OUTPUT lto_error)

add_executable(myapp main.cpp util.cpp math.cpp)

if(lto_supported)
    set_target_properties(myapp PROPERTIES
        INTERPROCEDURAL_OPTIMIZATION TRUE  # enables LTO
    )
    message(STATUS "LTO enabled")
else()
    message(WARNING "LTO not supported: ${lto_error}")
endif()

```

```bash

# Build without LTO:
cmake -B build-nolto -DCMAKE_BUILD_TYPE=Release
cmake --build build-nolto
ls -la build-nolto/myapp   # e.g., 245,000 bytes

# Build with LTO:
cmake -B build-lto -DCMAKE_BUILD_TYPE=Release
cmake --build build-lto
ls -la build-lto/myapp     # e.g., 198,000 bytes  (~20% smaller)

# Manual flags (GCC/Clang):
g++ -O2 -flto main.cpp util.cpp math.cpp -o myapp_lto
g++ -O2       main.cpp util.cpp math.cpp -o myapp_nolto

# ThinLTO (Clang only, faster builds):
clang++ -O2 -flto=thin main.cpp util.cpp math.cpp -o myapp_thinlto

```

### Q2: Full LTO vs ThinLTO

```cpp

Full LTO pipeline:
  a.cpp -> a.bc \                    
  b.cpp -> b.bc  > merge all IR -> single optimization pass -> codegen
  c.cpp -> c.bc /  (sequential, memory-intensive)

ThinLTO pipeline:
  a.cpp -> a.bc \   analyze    
  b.cpp -> b.bc  > thin link -> per-module optimization (parallel!) -> codegen
  c.cpp -> c.bc /   (summaries only, low memory)

```

| Aspect | Full LTO | ThinLTO |
| --- | --- | --- |
| Build speed | Slow (sequential) | Fast (parallel) |
| Memory usage | High (all IR loaded) | Low (summaries + on-demand) |
| Optimization quality | Best | 95-99% of full LTO |
| Incremental builds | None (must redo all) | Partial (only changed modules) |
| Best for | Small projects, final release | Large projects, CI |

```bash

# ThinLTO with parallelism:
clang++ -O2 -flto=thin -Wl,--thinlto-jobs=8 main.cpp util.cpp -o app

# CMake with ThinLTO:
set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} -flto=thin")
set(CMAKE_EXE_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE} -flto=thin")

# Cache ThinLTO results:
set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -Wl,--thinlto-cache-dir=thinlto-cache")

```

### Q3: Devirtualization with LTO

```cpp

// === shape.h ===
struct Shape {
    virtual int area() const = 0;
    virtual ~Shape() = default;
};

// === circle.cpp ===
#include "shape.h"
struct Circle : Shape {
    int r;
    Circle(int r) : r(r) {}
    int area() const override { return 3 * r * r; }  // simplified
};
Shape* make_circle(int r) { return new Circle(r); }

// === main.cpp ===
#include "shape.h"
#include <iostream>
extern Shape* make_circle(int r);

int main() {
    Shape* s = make_circle(5);
    int a = s->area();         // virtual call
    std::cout << a << '\n';
    delete s;
}

```

```bash

# Without LTO — virtual call remains:
$ g++ -O2 -c circle.cpp -o circle.o
$ g++ -O2 -c main.cpp -o main.o
$ g++ main.o circle.o -o app_nolto
# Assembly: call [vtable + offset]  <-- indirect call through vtable

# With LTO — devirtualized:
$ g++ -O2 -flto circle.cpp main.cpp -o app_lto
# Assembly: mov eax, 75   <-- computed at compile time (3*5*5=75)
#           No vtable lookup, no indirect call

# Why: LTO sees that make_circle always returns Circle,
# so area() always calls Circle::area(). The virtual call
# is devirtualized and then the function is inlined and
# constant-folded

```

---

## Notes

- LTO requires all objects compiled with the same `-flto` flag.
- GCC and Clang LTO object files are **not** interchangeable.
- Static libraries work with LTO; shared libraries have limited benefit.
- Debug info with LTO: use `-g -flto` (increases link time significantly).
- MSVC uses `/GL` (compile) + `/LTCG` (link) for its LTO equivalent.
