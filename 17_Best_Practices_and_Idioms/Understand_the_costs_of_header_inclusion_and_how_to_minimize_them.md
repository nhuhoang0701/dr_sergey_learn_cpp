# Understand the costs of header inclusion and how to minimize them

**Category:** Best Practices & Idioms  
**Item:** #403  
**Standard:** C++20  
**Reference:** <https://clang.llvm.org/docs/ClangCommandLineReference.html>  

---

## Topic Overview

Every `#include` copies the entire header file into every translation unit. Large headers (like `<iostream>`, `<algorithm>`, `<regex>`) can add thousands of lines, dramatically increasing compile time.

### Header Inclusion Costs

```cpp

main.cpp includes <iostream>     →  ~27,000 lines of code parsed
main.cpp includes <algorithm>    →  ~12,000 lines
main.cpp includes <regex>        →  ~50,000 lines
main.cpp includes "my_header.h"  →  depends on ITS includes!

If my_header.h includes <map>, <string>, <vector>,
then EVERY file that includes my_header.h pays that cost.

```

### The Include Graph Problem

```cpp

                a.h
               / | \
             b.h c.h d.h
            / \   |   \
          e.h f.h g.h  h.h

Changing e.h → recompiles everything that includes b.h → a.h

```

---

## Self-Assessment

### Q1: Use `-ftime-trace` in Clang to find the most expensive included headers

```cpp

# Step 1: Compile with -ftime-trace
$ clang++ -std=c++20 -O2 -ftime-trace main.cpp -o main

# This generates main.json (Chrome tracing format)

# Step 2: Open in chrome://tracing or https://ui.perfetto.dev/
# Or use ClangBuildAnalyzer:
$ ClangBuildAnalyzer --all . analysis.bin
$ ClangBuildAnalyzer --analyze analysis.bin

# Output shows:
# **** Templates that took longest to instantiate:
# **** Files that took longest to parse:
#   27439 ms: /usr/include/c++/13/regex
#   12891 ms: /usr/include/c++/13/iostream
#    8432 ms: /usr/include/c++/13/algorithm

```

```cpp

// Example: measure include cost
// heavy.cpp
#include <regex>       // ~50,000 lines, ~200ms parse time
#include <iostream>    // ~27,000 lines, ~80ms
#include <algorithm>   // ~12,000 lines, ~40ms
#include <string>      // ~8,000 lines, ~25ms

int main() {
    std::string s = "hello";
    std::cout << s << '\n';
    return 0;
}
// Only needs <string> and <iostream>, but <regex> and <algorithm>
// add ~320ms of needless parse time per TU!

```

### Q2: Replace full includes with forward declarations

```cpp

// BAD: widget.h includes everything
// widget_bad.h
#include <string>     // full include (needed for member)
#include <vector>     // full include (needed for member)
#include "logger.h"   // full include, but we only use Logger*!
#include "database.h" // full include, but we only use Database&!

class WidgetBad {
    std::string name_;
    std::vector<int> data_;
    Logger* logger_;         // pointer: only needs forward declaration
    Database& db_;           // reference: only needs forward declaration
public:
    WidgetBad(Database& db, Logger* log);
    void process();
};

// GOOD: widget.h with forward declarations
// widget_good.h
#include <string>     // needed: member variable
#include <vector>     // needed: member variable

class Logger;          // forward declaration (pointer in class)
class Database;        // forward declaration (reference in class)

class WidgetGood {
    std::string name_;
    std::vector<int> data_;
    Logger* logger_;
    Database& db_;
public:
    WidgetGood(Database& db, Logger* log);
    void process();
};

// widget_good.cpp — include the full headers HERE
// #include "widget_good.h"
// #include "logger.h"     // now we need the full definition
// #include "database.h"   // for calling methods

```

**When you CAN use forward declarations:**
| Usage of type T | Forward decl OK? |
| --- | --- |
| `T*`, `T&`, `const T&` | Yes |
| `std::unique_ptr<T>` (in header) | Yes (if destructor in .cpp) |
| `T` as member variable | **No** (need sizeof) |
| `T` as base class | **No** |
| Calling `T::method()` | **No** |
| `T` in template parameter | Depends |

### Q3: Explain how IWYU and module migration reduce cascading include costs

**IWYU (Include What You Use):**

```cpp

# Install and run:
$ apt install iwyu
$ iwyu_tool.py -p build/ main.cpp

# Output:
main.cpp should add:
  #include <string_view>
main.cpp should remove:
  #include <algorithm>   // not directly used
  #include <map>         // not directly used

```

IWYU ensures each file includes **only what it directly uses**, breaking unnecessary transitive dependencies.

**C++20 Modules:**

```cpp

// math.cppm (module interface)
export module math;

export int add(int a, int b) { return a + b; }
export int multiply(int a, int b) { return a * b; }
// Internal helpers are NOT exported
int internal_helper() { return 42; }  // invisible to importers

// main.cpp
import math;  // only sees exported symbols
              // does NOT re-parse the entire module every time
              // compiled ONCE into a BMI (Binary Module Interface)

int main() {
    return add(1, 2);
}

```

| Feature | `#include` | `import` (modules) |
| --- | --- | --- |
| Copies text | Yes (every TU) | No (compiled once) |
| Macro leakage | Yes | No |
| Order dependent | Yes | No |
| Build speedup | Baseline | 2-5x faster |

---

## Notes

- Precompiled headers (PCH) are the pre-C++20 solution: compile common headers once.
- Use `#pragma once` or include guards to prevent double-inclusion.
- The "Pimpl idiom" is the ultimate forward-declaration technique (all implementation behind a pointer).
- Header-only libraries (like Boost.Spirit) can devastate compile times.
- GCC's `-H` flag prints the include tree; `-ftime-report` shows compilation phase timing.
