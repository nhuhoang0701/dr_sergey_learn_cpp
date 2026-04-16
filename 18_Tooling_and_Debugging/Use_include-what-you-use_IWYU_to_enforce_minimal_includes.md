# Use include-what-you-use (IWYU) to enforce minimal includes

**Category:** Tooling & Debugging  
**Item:** #224  
**Reference:** <https://include-what-you-use.org>  

---

## Topic Overview

IWYU analyzes your `#include` directives and reports:

- Headers you include but don't need (remove them).
- Headers you use transitively but should include directly.

```cpp

Before IWYU:                       After IWYU:
#include <algorithm>               #include <algorithm>  // for std::sort
#include <iostream>                #include <vector>     // for std::vector
#include <vector>                  #include <string>     // for std::string
#include <map>         // UNUSED!
#include <functional>  // UNUSED!
// std::string used but not        // <iostream> removed (unused)
// directly included!              // <map>, <functional> removed

```

---

## Self-Assessment

### Q1: Run IWYU and remove transitive includes

```cpp

// bad_includes.cpp — relies on transitive includes
#include <algorithm>   // pulls in <utility> on some implementations
#include <map>         // not used at all!

// Uses std::string but doesn't include <string>
// Works on GCC because <algorithm> transitively includes it.
void greet(const std::string& name) {
    // Uses std::vector but doesn't include <vector>
    // Works because <algorithm> pulls it in on this compiler.
    std::vector<int> data = {3, 1, 2};
    std::sort(data.begin(), data.end());
}

int main() {
    greet("World");
}

```

```bash

# Run IWYU:
$ include-what-you-use -std=c++20 bad_includes.cpp

# IWYU output:
# bad_includes.cpp should add these lines:
#   #include <string>   // for std::string
#   #include <vector>   // for std::vector
#
# bad_includes.cpp should remove these lines:
#   - #include <map>    // lines 2-2
#
# The full include-list for bad_includes.cpp:
#   #include <algorithm>  // for sort
#   #include <string>     // for string
#   #include <vector>     // for vector

# Auto-apply fixes:
$ include-what-you-use -std=c++20 bad_includes.cpp 2>&1 | fix_includes.py

```

Fixed version:

```cpp

// bad_includes.cpp — after IWYU
#include <algorithm>   // for std::sort
#include <string>      // for std::string
#include <vector>      // for std::vector

void greet(const std::string& name) {
    std::vector<int> data = {3, 1, 2};
    std::sort(data.begin(), data.end());
}

int main() {
    greet("World");
}

```

### Q2: Why transitive includes are fragile

```cpp

// This code compiles on GCC 12 but FAILS on GCC 13:
#include <algorithm>
// GCC 12: <algorithm> includes <string> internally
// GCC 13: <algorithm> no longer includes <string>

void process(const std::string& s) {  // ERROR on GCC 13!
    // std::string not declared
}

```

```cpp

Transitive include fragility:

GCC 12:                              GCC 13:
<algorithm>                          <algorithm>
  └─> <utility>                        └─> <utility>
        └─> <string>  ✓ works!               (no <string>)  ✗ breaks!

The fix: always #include what you directly use.

```

Real-world examples of broken transitive includes:

- GCC 13 removed `<string>`, `<cstdint>`, `<algorithm>` transitive includes from many headers.
- libc++ (Clang) has always been stricter than libstdc++ about transitive includes.
- Upgrading compilers without IWYU frequently breaks builds.

### Q3: Configure IWYU mappings for custom libraries

```yaml

# my_library.imp — IWYU mapping file
[
  # Map internal headers to public headers:
  { include: ["\"mylib/internal/impl.h\"", private,
              "\"mylib/mylib.h\"", public] },

  # Map multiple internals to one public header:
  { include: ["\"mylib/internal/parser_impl.h\"", private,
              "\"mylib/parser.h\"", public] },
  { include: ["\"mylib/internal/lexer_impl.h\"", private,
              "\"mylib/parser.h\"", public] },

  # Symbol-based mapping:
  { symbol: ["mylib::Widget", private,
             "\"mylib/widget.h\"", public] },

  # Reference an existing mapping file:
  { ref: "third_party/boost.imp" }
]

```

```bash

# Use mapping file:
$ include-what-you-use -Xiwyu --mapping_file=my_library.imp -std=c++20 main.cpp

# CMake integration:
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_INCLUDE_WHAT_YOU_USE
    include-what-you-use
    -Xiwyu --mapping_file=${CMAKE_SOURCE_DIR}/my_library.imp
    -Xiwyu --no_fwd_decls  # optional: disable forward declaration suggestions
)

add_executable(myapp main.cpp)

```

---

## Notes

- IWYU uses Clang's frontend; pass Clang-compatible flags.
- `--no_fwd_decls` disables forward declaration suggestions (sometimes too aggressive).
- Mapping files handle internal/public header distinctions.
- Run IWYU in CI to prevent transitive include regressions.
- IWYU may not support the very latest C++23/26 features yet.
