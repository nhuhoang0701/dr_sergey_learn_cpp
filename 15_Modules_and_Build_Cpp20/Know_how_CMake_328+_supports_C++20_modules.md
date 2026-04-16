# Know how CMake 3.28+ supports C++20 modules

**Category:** Modules & Build (C++20)  
**Item:** #517  
**Standard:** C++20  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-cxxmodules.7.html>  

---

## Topic Overview

CMake 3.28 introduced **native C++20 module support** via the `CXX_MODULES` file set type. Before 3.28, module support was experimental (requiring `CMAKE_EXPERIMENTAL_CXX_MODULE_DYNDEP`).

### Key CMake Features for Modules

| Feature | CMake Version | Purpose |
| --- | --- | --- |
| `FILE_SET CXX_MODULES` | 3.28+ | Declare module interface files |
| Dependency scanning | 3.28+ | Automatic `import` scanning (P1689R5) |
| Module BMI caching | 3.28+ | Reuse compiled module interfaces |
| `import std` | 3.30+ | Standard library module import |
| Generator support | Ninja 1.11+, MSBuild 17.4+ | Required build generators |

### Build Order: Headers vs Modules

```cpp

HEADER-BASED BUILD (all TUs compile in parallel):
  [a.cpp] ───┐
  [b.cpp] ───┤── (parallel) ──> link
  [c.cpp] ───┘

MODULE-BASED BUILD (dependency scanning creates order):
  [my_math.cppm]  ──> compile module interface ──┐
  [main.cpp]      ────── (waits for module) ──────┴──> link
  [utils.cpp]     ────── (parallel, no module dep) ─┘

```

Modules require the build system to **scan for `import` declarations** before compiling, creating a DAG (directed acyclic graph) of module dependencies.

---

## Self-Assessment

### Q1: Write a CMakeLists.txt using `target_sources` with `FILE_SET CXX_MODULES` for a module

**Project structure:**

```cpp

project/
├── CMakeLists.txt
├── src/
│   ├── math.cppm          # module interface unit
│   ├── math_impl.cpp      # module implementation unit
│   └── main.cpp

```

**CMakeLists.txt:**

```cmake

cmake_minimum_required(VERSION 3.28)
project(ModuleDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

add_executable(app)

target_sources(app
  PRIVATE
    src/main.cpp
    src/math_impl.cpp
)

# Declare module interface files with FILE_SET CXX_MODULES
target_sources(app
  PUBLIC
    FILE_SET CXX_MODULES FILES
      src/math.cppm
)

```

**src/math.cppm** (module interface):

```cpp

export module math;

export int add(int a, int b);
export int multiply(int a, int b);

export namespace geometry {
    double circle_area(double radius);
}

```

**src/math_impl.cpp** (module implementation):

```cpp

module math;

import <cmath>;  // or #include <cmath> if header units unsupported

int add(int a, int b) { return a + b; }
int multiply(int a, int b) { return a * b; }

namespace geometry {
    double circle_area(double radius) { return std::acos(-1.0) * radius * radius; }
}

```

**src/main.cpp:**

```cpp

import math;
#include <iostream>

int main() {
    std::cout << "3 + 4 = " << add(3, 4) << '\n';
    std::cout << "3 * 4 = " << multiply(3, 4) << '\n';
    std::cout << "Circle area r=5: " << geometry::circle_area(5.0) << '\n';
}
// Expected output:
// 3 + 4 = 7
// 3 * 4 = 12
// Circle area r=5: 78.5398

```

**Key points:**

- `FILE_SET CXX_MODULES` tells CMake these `.cppm` files are module interfaces.
- CMake scans these files for `export module ...` and `import ...` to build the dependency graph.
- Use Ninja 1.11+ or Visual Studio 17.4+ as the generator.

### Q2: Explain the build system's dependency scanning requirement for modules vs textual includes

**Headers (textual includes):**

```cpp

Step 1: Parse source files       ← no scanning needed
Step 2: Compile ALL .cpp in parallel ← preprocessor handles #include during compilation
Step 3: Link

The compiler resolves #include during preprocessing — no build system involvement.

```

**Modules:**

```cpp

Step 1: SCAN all source files for import/export declarations (P1689R5 format)
Step 2: Build dependency DAG from scan results
Step 3: Compile module interfaces in topological order
Step 4: Compile implementation units (can use compiled module interfaces)
Step 5: Link

The build system MUST know module dependencies BEFORE compilation.

```

**Why scanning is necessary:**

```cpp

// file: main.cpp
import math;   // The build system must know:
               //  1. Which file provides "module math"?
               //  2. Is that module already compiled?
               //  3. Where is the BMI (binary module interface)?

```

| Aspect | `#include` (headers) | `import` (modules) |
| --- | --- | --- |
| Resolution | Compiler, during preprocessing | Build system, before compile |
| Dependency info | Implicit (include paths) | Explicit (scanned declarations) |
| Build order | All TUs independent | Module interfaces must compile first |
| Rebuild trigger | File timestamp | Module interface change → recompile importers |
| Standard format | None | P1689R5 JSON dependency format |

### Q3: Show the difference in build parallelism between header-based and module-based builds

**Header-based: Maximum parallelism**

```cpp

Time  →  [=====]  a.cpp        All 4 files compile
         [=====]  b.cpp        simultaneously.
         [=====]  c.cpp        Each re-parses every
         [=====]  d.cpp        #included header.
               [==] LINK

Total: 5 + 2 = 7 time units
Redundant work: each TU parses <vector>, <string>, etc. independently

```

**Module-based: Ordered but less redundant work**

```cpp

Time  →  [==]  math.cppm      Module interface compiles first
              [=] b.cpp        Importers wait for module BMI,
              [=] c.cpp        but compile faster (no re-parsing)
              [=] d.cpp        
                  [==] LINK

Total: 2 + 1 + 2 = 5 time units
Module BMI compiled once, reused by all importers

```

**Key tradeoffs:**

| Metric | Headers | Modules |
| --- | --- | --- |
| Parallelism | All TUs independent | DAG-constrained |
| Per-TU compile time | Slower (re-parses headers) | Faster (reads BMI) |
| Total build time | Depends on redundant parsing | Depends on module DAG depth |
| Incremental rebuild | Recompile changed TU only | Recompile module + all importers |
| Clean build | O(N × H) parsing work | O(N + M) parsing work |

- N = number of translation units, H = average headers per TU, M = number of modules
- Modules win on **clean builds** of large projects because header re-parsing dominates.
- For small projects, the scanning overhead may negate the benefit.

---

## Notes

- Use `.cppm`, `.ixx` (MSVC), or `.mpp` extensions for module interface files — CMake recognizes these.
- Set `CMAKE_CXX_SCAN_FOR_MODULES ON` (default in 3.28+ for C++20 targets).
- Ninja is strongly recommended over Make for module builds (Make lacks dynamic dependency support).
- Module support varies by compiler: MSVC 17.4+ (best), GCC 14+, Clang 16+ (partial).
