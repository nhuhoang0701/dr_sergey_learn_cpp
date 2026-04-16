# Control symbol visibility with attributes and declspec for shared libraries

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://gcc.gnu.org/wiki/Visibility>  

---

## Topic Overview

By default, all symbols in a shared library are exported. This causes: bloated symbol tables, slow load times, symbol clashes, and ABI fragility. Controlling visibility exports only the public API.

### Cross-Platform Visibility Macro

```cpp

#pragma once

#if defined(_WIN32)
  #ifdef MYLIB_EXPORTS
    #define MYLIB_API __declspec(dllexport)
  #else
    #define MYLIB_API __declspec(dllimport)
  #endif
  #define MYLIB_LOCAL
#else
  #if __GNUC__ >= 4
    #define MYLIB_API   __attribute__((visibility("default")))
    #define MYLIB_LOCAL  __attribute__((visibility("hidden")))
  #else
    #define MYLIB_API
    #define MYLIB_LOCAL
  #endif
#endif

// Usage:
namespace mylib {
    MYLIB_API void public_function();   // Exported
    MYLIB_LOCAL void internal_helper(); // Hidden — not in symbol table

    class MYLIB_API PublicClass {
    public:
        void method();         // Exported (class is exported)
    private:
        MYLIB_LOCAL void impl(); // Hidden even in exported class
    };
}

```

### CMake Integration

```cmake

add_library(mylib SHARED src/api.cpp src/internal.cpp)

# -fvisibility=hidden makes all symbols hidden by default
# Only MYLIB_API-marked symbols are exported
target_compile_options(mylib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-fvisibility=hidden -fvisibility-inlines-hidden>
)

target_compile_definitions(mylib PRIVATE MYLIB_EXPORTS)

```

---

## Self-Assessment

### Q1: Measure the symbol table reduction from visibility control

```bash

# Before: all symbols exported
nm -D libmylib.so | wc -l    # e.g., 2500 symbols

# After: -fvisibility=hidden + explicit MYLIB_API
nm -D libmylib.so | wc -l    # e.g., 45 symbols

# Benefits:
# - ~98% fewer exported symbols
# - Faster dynamic linking (fewer symbols to resolve)
# - No symbol clashes between libraries
# - Compiler can optimize hidden functions more aggressively

```

### Q2: Why use `-fvisibility-inlines-hidden`

Inline functions and template instantiations generate many symbols. With `-fvisibility-inlines-hidden`, these are hidden by default, preventing them from polluting the shared library's symbol table. This is almost always what you want — inline functions should be resolved at compile time, not link time.

### Q3: How to export a class template from a shared library

```cpp

// Explicit instantiation with export
template class MYLIB_API std::vector<MyType>; // Export specific instantiation

// Template DEFINITIONS stay in headers
// Only explicit instantiations are exported from the .so

```

---

## Notes

- Always compile shared libraries with `-fvisibility=hidden` and mark public API explicitly.
- On Windows, `__declspec(dllexport/dllimport)` is required; there's no "hidden by default" mode.
- `__attribute__((visibility("default")))` on GCC/Clang is equivalent to `dllexport`.
- Exception types thrown across library boundaries MUST be visible (exported).
