# Understand how to mix modules with header-based code using header units

**Category:** Modules & Build (C++20)  
**Item:** #123  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

**Header units** are a transitional mechanism between `#include` and modules. They compile a header into a module-like format that can be imported with `import <header>;` syntax.

### The Spectrum from Headers to Modules

```cpp

┌────────────────┐    ┌────────────────┐    ┌────────────────┐
│  #include       │ →  │  import <hdr>;  │ →  │  import module; │
│  (textual)      │    │  (header unit)  │    │  (named module) │
│                 │    │                 │    │                 │
│ • macros leak   │    │ • macros export │    │ • no macro leak │
│ • reparsed/TU   │    │ • compiled once │    │ • compiled once │
│ • no dep info   │    │ • module-like   │    │ • full isolation│
└────────────────┘    └────────────────┘    └────────────────┘
    Legacy              Transitional           Modern

```

### Comparison Table

| Feature | `#include <vector>` | `import <vector>;` | `import std;` |
| --- | --- | --- | --- |
| Parse cost | Every TU | Once | Once |
| Macros visible | Yes | Yes (!) | No |
| Available since | Always | C++20 | C++23 |
| Compiler support | All | MSVC, GCC, Clang | MSVC 17.5+ |
| Migration effort | None | Minimal | Moderate |

---

## Self-Assessment

### Q1: Import a legacy header as a header unit using `import <vector>;`

```cpp

// Using import <header> syntax for header units
import <iostream>;
import <vector>;
import <algorithm>;
import <string>;

int main() {
    // All standard library features available via header unit imports
    std::vector<std::string> names = {"Charlie", "Alice", "Bob"};
    std::sort(names.begin(), names.end());

    for (const auto& name : names)
        std::cout << name << ' ';
    std::cout << '\n';

    // Macros from header units ARE visible (unlike named modules)
    // <cassert> defines the assert macro
    // import <cassert>;  // assert() macro would be available
}
// Expected output:
// Alice Bob Charlie

```

**Compilation (MSVC):**

```bash

# MSVC needs /headerUnit for each header unit
cl /std:c++20 /EHsc /headerUnit:angle <vector> <algorithm> main.cpp

# Or use /translateInclude to auto-convert #include to import
cl /std:c++20 /EHsc /translateInclude main.cpp

```

**Compilation (GCC):**

```bash

# GCC uses -fmodule-header for header units
g++ -std=c++20 -fmodules-ts -fmodule-header=<vector> main.cpp

```

**Key point:** `import <vector>;` compiles `<vector>` into a header unit once, then reuses it across all TUs — faster than re-parsing every time.

### Q2: Explain the difference between `import <header>` (header unit) and `#include <header>`

| Behavior | `#include <header>` | `import <header>;` |
| --- | --- | --- |
| **Processing** | Textual copy-paste into TU | Read pre-compiled header unit |
| **Macros** | Injected into TU | Also injected (header units export macros!) |
| **Parse time** | O(header_size) per TU | O(1) per TU (read BMI) |
| **ODR safety** | Fragile (different define contexts) | Safer (single compilation) |
| **Include order** | Matters (macros affect later includes) | Independent of order |
| **Preprocessor state** | Changed by macros from header | Changed by macros from header unit |

**Important subtlety — macros and header units:**

```cpp

// With #include:
#define _LIBCPP_DEBUG 1
#include <vector>           // picks up debug mode via macro

// With import:
#define _LIBCPP_DEBUG 1
import <vector>;            // the header unit was compiled WITHOUT this macro!
                            // The define has no effect on the import

```

Header units are compiled in a **fixed macro environment** — macros defined by the importer before `import` do NOT affect the header unit's compilation. This is fundamentally different from `#include`.

### Q3: Show how to gradually migrate a codebase from headers to modules

**Phase 1: No changes (baseline)**

```cpp

// math_utils.h
#pragma once
#include <cmath>

inline double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp
#include "math_utils.h"
#include <iostream>

int main() {
    std::cout << distance(3.0, 4.0) << '\n';  // 5
}

```

**Phase 2: Switch standard includes to header units**

```cpp

// math_utils.h (unchanged)
#pragma once
#include <cmath>

inline double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp — use header unit for standard header
#include "math_utils.h"  // still #include for project header
import <iostream>;       // switch to header unit for std headers

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}

```

**Phase 3: Convert project headers to modules**

```cpp

// math_utils.cppm — now a module
module;
#include <cmath>  // C header in global module fragment

export module math_utils;

export double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp — import the module
import math_utils;
import <iostream>;

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}

```

**Phase 4: Use `import std;` (C++23)**

```cpp

// math_utils.cppm
export module math_utils;
import std;  // replaces <cmath> include

export double distance(double x, double y) {
    return std::sqrt(x * x + y * y);
}

// main.cpp
import math_utils;
import std;

int main() {
    std::cout << distance(3.0, 4.0) << '\n';
}

```

**Migration strategy summary:**

| Phase | Change | Risk | Benefit |
| --- | --- | --- | --- |
| 1 | Baseline | None | None |
| 2 | `import <header>` for std | Low | Faster std header parsing |
| 3 | Convert own headers to modules | Medium | Macro isolation, better encapsulation |
| 4 | `import std;` | Low | Simplest possible includes |

---

## Notes

- Header units and named modules can coexist in the same project — use header units for libraries not yet modularized.
- Not all headers can be header units (headers with complex macro interactions may fail).
- MSVC `/translateInclude` flag can auto-convert `#include` to header unit imports for standard headers.
- Header units are **not** the same as precompiled headers (PCH) — they have module semantics and proper isolation.
