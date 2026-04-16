# Use CMake module dependency scanning for C++20 modules

**Category:** Build Systems & CI  
**Item:** #567  
**Standard:** C++20  
**Reference:** <https://cmake.org/cmake/help/latest/prop_tgt/CXX_SCAN_FOR_MODULES.html>  

---

## Topic Overview

C++20 modules break the traditional `#include`-based compilation model. Unlike headers, modules must be **compiled before** their importers — the build system must discover `import` dependencies **before** deciding compilation order. CMake 3.28+ supports this via **module dependency scanning**.

### Two-Phase Build Model

```cpp

Traditional (#include):
  src1.cpp ──compile──► src1.o ─┐
  src2.cpp ──compile──► src2.o ─┼─► link ──► app
  src3.cpp ──compile──► src3.o ─┘
  (all TUs compile in parallel — no ordering constraint)

C++20 Modules:
  Phase 1: Scan all sources for import/export declarations
    math.cppm   → exports "math"
    main.cpp    → imports "math"
    
  Phase 2: Compile in dependency order
    math.cppm ──compile──► math.pcm + math.o
    main.cpp  ──compile──► main.o   (needs math.pcm)
    link: main.o + math.o ──► app

```

### Compiler Support Status

| Compiler | Module Support | CMake Scanning |
| --- | --- | --- |
| GCC 14+ | Good | CMake 3.28+ |
| Clang 16+ | Good | CMake 3.28+ |
| MSVC 17.4+ | Good | CMake 3.28+ |

---

## Self-Assessment

### Q1: Enable CXX_SCAN_FOR_MODULES in CMake 3.28+ and verify correct build ordering for module TUs

**Answer:**

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.28)    # Module scanning requires 3.28+
project(ModulesDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ═══════════ Create a library with module sources ═══════════
add_library(mathlib)
target_sources(mathlib
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/math.cppm           # Module interface unit
            src/math_impl.cppm      # Module implementation partition
)
# CXX_SCAN_FOR_MODULES is ON by default for targets with CXX_MODULES file set

# ═══════════ Executable that imports the module ═══════════
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mathlib)

# ═══════════ Verify scanning is enabled ═══════════
# CMake automatically sets CXX_SCAN_FOR_MODULES=ON when you use
# FILE_SET CXX_MODULES. You can check explicitly:
get_target_property(scan_enabled mathlib CXX_SCAN_FOR_MODULES)
message(STATUS "Module scanning enabled: ${scan_enabled}")

```

```cpp

// src/math.cppm — Module interface unit
export module math;

export namespace math {
    constexpr double pi = 3.14159265358979323846;
    
    constexpr double square(double x) { return x * x; }
    constexpr double cube(double x)   { return x * x * x; }
    double circle_area(double radius);
}

```

```cpp

// src/math_impl.cppm — Module implementation unit
module math;

namespace math {
    double circle_area(double radius) {
        return pi * square(radius);
    }
}

```

```cpp

// src/main.cpp
import math;
#include <iostream>

int main() {
    std::cout << "pi = " << math::pi << '\n';
    std::cout << "5² = " << math::square(5) << '\n';
    std::cout << "Area of circle(r=3) = " << math::circle_area(3) << '\n';
}

```

```bash

# Build with Ninja (recommended for module builds):
cmake -B build -G Ninja
cmake --build build --verbose
# Output shows dependency-ordered compilation:
# [1/4] Scanning src/math.cppm for CXX dependencies
# [2/4] Building CXX object math.cppm.o        ← compiled first
# [3/4] Building CXX object main.cpp.o          ← compiled after math.pcm exists
# [4/4] Linking CXX executable app

```

### Q2: Explain the two-phase build: dependency scanning pass then compilation in dependency order

**Answer:**

```cpp

Phase 1: SCANNING (p1689 format)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
For each source file, the compiler produces a .ddi file (dep info):

  clang-scan-deps -format=p1689 -- clang++ -std=c++20 math.cppm
  → { "provides": [{"logical-name": "math"}], "requires": [] }

  clang-scan-deps -format=p1689 -- clang++ -std=c++20 main.cpp
  → { "provides": [], "requires": [{"logical-name": "math"}] }

Phase 2: BUILD GRAPH CONSTRUCTION
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
Ninja/CMake reads all .ddi files and constructs a DAG:

  math.cppm ──(provides "math")──► math.pcm
       ↑                               ↓
       │                         main.cpp (requires "math")
       │                               ↓
       └───────────────────────────── link

Phase 3: COMPILATION (dependency-ordered)
━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━━
  Step 1: Compile math.cppm → math.o + math.pcm (BMI file)
  Step 2: Compile main.cpp  → main.o  (uses math.pcm)
  Step 3: Link math.o + main.o → app

```

**Key terms:**

- **BMI (Binary Module Interface):** `.pcm` (Clang), `.ifc` (MSVC), `.gcm` (GCC) — the compiled module interface
- **P1689:** Standard JSON format for describing module dependencies
- **`.ddi`:** Dependency discovery information file (CMake internal)
- **DAG:** Directed acyclic graph — determines compilation order

**Why Ninja is preferred over Make:**

```bash

# Ninja supports dynamic dependencies (dyndep) — can adjust build
# order AFTER reading scan results. Make cannot do this
cmake -B build -G Ninja     # ✅ Best for modules
cmake -B build -G "Unix Makefiles"  # ⚠️ Limited module support

```

### Q3: Show the CMakeLists.txt changes needed to add a .cppm module file to a target

**Answer:**

```cmake

# ═══════════ Before modules (traditional headers) ═══════════
add_library(mathlib
    src/math.cpp
    src/math.h           # header — no special treatment needed
)
target_include_directories(mathlib PUBLIC include/)

# ═══════════ After: adding module interface units ═══════════
cmake_minimum_required(VERSION 3.28)   # MUST be 3.28+

add_library(mathlib)

# Step 1: Add module interface files via FILE_SET CXX_MODULES
target_sources(mathlib
    PUBLIC
        FILE_SET CXX_MODULES FILES
            src/math.cppm                # Primary module interface
            src/math-utils.cppm          # Module partition interface
    PRIVATE
        src/math_impl.cpp               # Regular TU (can import math)
)

# Step 2: That's it! CMake automatically:
#   - Sets CXX_SCAN_FOR_MODULES=ON
#   - Scans for import/export declarations
#   - Builds .cppm files before their dependents
#   - Installs BMI files alongside headers

# ═══════════ Consuming from another target ═══════════
add_executable(app src/main.cpp)
target_link_libraries(app PRIVATE mathlib)
# main.cpp can now write: import math;

# ═══════════ Module partitions ═══════════
# src/math.cppm:
#   export module math;
#   export import :utils;    // re-export partition
#
# src/math-utils.cppm:
#   export module math:utils;
#   export int add(int a, int b) { return a + b; }
#
# Both partition and primary interface go in FILE_SET CXX_MODULES

# ═══════════ Common mistakes ═══════════
# ✗ Don't add .cppm to target_sources() without FILE_SET:
#     add_library(mathlib src/math.cppm)  # WRONG — no scanning
#
# ✗ Don't forget cmake_minimum_required(VERSION 3.28):
#     VERSION 3.20 won't enable scanning
#
# ✗ Don't use Makefiles:
#     -G "Unix Makefiles"  # Limited module support — use Ninja

```

---

## Notes

- **File extensions:** `.cppm` (Clang convention), `.ixx` (MSVC convention), `.ccm` (GCC) — CMake recognizes all when added via `FILE_SET CXX_MODULES`
- **Header units** (`import <iostream>;`) are not yet well supported in CMake — stick with `#include` for standard headers
- **`CMAKE_CXX_MODULE_STD`** (CMake 3.30+): enables `import std;` support for the standard library module
- **Install:** `install(TARGETS mathlib FILE_SET CXX_MODULES DESTINATION lib/cmake/mathlib)` installs BMI files for downstream consumers
