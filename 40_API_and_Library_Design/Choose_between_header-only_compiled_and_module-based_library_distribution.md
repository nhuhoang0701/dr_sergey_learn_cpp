# Choose between header-only, compiled, and module-based library distribution

**Category:** API & Library Design  
**Standard:** C++20  
**Reference:** <https://isocpp.org/wiki/faq/cpp20-library-design>  

---

## Topic Overview

### Comparison

| Aspect | Header-Only | Compiled (.a/.lib) | C++20 Modules |
| --- | --- | --- | --- |
| Integration | `#include` only | Link step required | `import` |
| Build speed | Slow (recompiled per TU) | Fast (compiled once) | Fast |
| Template support | Full | Explicit instantiation needed | Full |
| ABI coupling | High (inline everything) | Low (stable .so) | Medium |
| Ease of use | Simplest | Moderate | Moderate |
| Example | nlohmann/json, Catch2 | Boost.Filesystem, OpenSSL | std module |

### When to Choose Each

```cpp

Header-only:
  ✓ Mostly templates / constexpr
  ✓ Small library (< 10K LOC)
  ✓ Users want zero build config
  ✗ Slow compile times for large libraries

Compiled library:
  ✓ Non-template code
  ✓ ABI stability needed (shared library)
  ✓ Large codebase
  ✗ Users must link correctly

C++20 Modules:
  ✓ Best of both worlds (templates + fast build)
  ✓ New projects with modern compiler requirements
  ✗ Toolchain support still maturing (2026)

```

### Header-Only Library Pattern

```cpp

// mylib.hpp — entire library in one header
#ifndef MYLIB_HPP
#define MYLIB_HPP

#include <string>
#include <vector>

namespace mylib {

// Templates are naturally header-only
template<typename T>
T clamp(T val, T lo, T hi) {
    return val < lo ? lo : val > hi ? hi : val;
}

// Non-template functions: use inline to avoid ODR violations
inline std::string trim(std::string_view sv) {
    auto start = sv.find_first_not_of(" \t\n\r");
    auto end = sv.find_last_not_of(" \t\n\r");
    if (start == std::string_view::npos) return {};
    return std::string(sv.substr(start, end - start + 1));
}

} // namespace mylib
#endif

```

### Compiled Library Pattern

```cpp

// mylib/api.hpp — public header
#pragma once
#include <string>

#ifdef MYLIB_BUILDING
  #define MYLIB_API __declspec(dllexport)  // or __attribute__((visibility("default")))
#else
  #define MYLIB_API __declspec(dllimport)
#endif

namespace mylib {
    MYLIB_API std::string process(const std::string& input);
    MYLIB_API int version();
}

// mylib/api.cpp — compiled once, linked
#define MYLIB_BUILDING
#include "api.hpp"

namespace mylib {
    std::string process(const std::string& input) {
        // Implementation hidden from users
        return "processed: " + input;
    }
    int version() { return 2; }
}

```

---

## Self-Assessment

### Q1: What are the ODR risks with header-only libraries

Every translation unit that includes the header gets its own copy of all functions. Non-inline, non-template functions in headers violate the One Definition Rule. Fix: mark all non-template functions `inline`, or use anonymous namespaces for internal helpers (but this creates separate copies).

### Q2: How do modules solve the header-only build time problem

Modules are compiled once into a Binary Module Interface (BMI). When 100 source files `import mylib;`, the module is parsed and type-checked once, not 100 times. This eliminates redundant parsing, macro expansion, and template instantiation that `#include` causes.

### Q3: Show a hybrid approach: header-only with optional compiled mode

```cpp

// config.hpp
#ifdef MYLIB_HEADER_ONLY
  #define MYLIB_INLINE inline
#else
  #define MYLIB_INLINE
#endif

// api.hpp
#pragma once
#include "config.hpp"

namespace mylib {
    MYLIB_INLINE std::string process(const std::string& input);
}

#ifdef MYLIB_HEADER_ONLY
  #include "api_impl.hpp" // Include implementation
#endif

// Users choose: -DMYLIB_HEADER_ONLY or link libmylib.a

```

---

## Notes

- Most modern C++ libraries (fmt, spdlog, Eigen) offer both header-only and compiled modes.
- C++20 modules are the future but require CMake 3.28+ and recent compilers.
- Header-only libraries are easier for package managers (vcpkg, Conan) to distribute.
