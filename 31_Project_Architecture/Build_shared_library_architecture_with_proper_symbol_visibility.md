# Build shared library architecture with proper symbol visibility

**Category:** Project Architecture

---

## Topic Overview

**Symbol visibility** controls which functions and classes are accessible from outside a shared library (`.so`/`.dll`). By default, GCC exports everything; MSVC exports nothing. Proper visibility reduces library size, improves load times, prevents symbol collisions, and enforces a clean public API boundary.

### Visibility Defaults

| Compiler | Default | Result |
| --- | --- | --- |
| GCC/Clang (Linux) | All symbols visible | Everything exported (bad) |
| MSVC (Windows) | Nothing exported | Must `__declspec(dllexport)` each symbol |
| GCC with `-fvisibility=hidden` | All hidden | Must explicitly mark exports |

---

## Self-Assessment

### Q1: Implement cross-platform export macros

**Answer:**

```cpp

// === mylib_export.h — cross-platform visibility macros ===
#pragma once

#if defined(_MSC_VER)
    // MSVC: dllexport when building, dllimport when consuming
    #ifdef MYLIB_BUILDING
        #define MYLIB_API __declspec(dllexport)
    #else
        #define MYLIB_API __declspec(dllimport)
    #endif
    #define MYLIB_HIDDEN
#elif defined(__GNUC__) || defined(__clang__)
    // GCC/Clang: use visibility attributes
    #define MYLIB_API __attribute__((visibility("default")))
    #define MYLIB_HIDDEN __attribute__((visibility("hidden")))
#else
    #define MYLIB_API
    #define MYLIB_HIDDEN
#endif

// === CMakeLists.txt ===
// add_library(mylib SHARED src/mylib.cpp)
// target_compile_definitions(mylib PRIVATE MYLIB_BUILDING)
// if(CMAKE_CXX_COMPILER_ID MATCHES "GNU|Clang")
//     target_compile_options(mylib PRIVATE -fvisibility=hidden)
// endif()

// === Using the macros ===
// mylib.h
#include "mylib_export.h"

namespace mylib {

// Exported: visible to consumers
class MYLIB_API Widget {
public:
    Widget();
    void do_something();
    int get_value() const;

private:
    // Private members are part of the binary layout but not the API
    class Impl;  // PIMPL hides implementation details
    std::unique_ptr<Impl> impl_;
};

MYLIB_API void initialize();
MYLIB_API void shutdown();
MYLIB_API const char* version();

// NOT exported: internal helper
MYLIB_HIDDEN void internal_helper();

}  // namespace mylib

```

### Q2: Use CMake's GenerateExportHeader for automated macros

**Answer:**

```cmake

# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(mylib VERSION 1.0.0)

add_library(mylib SHARED
    src/widget.cpp
    src/init.cpp
)

# Automatically generate export macros
include(GenerateExportHeader)
generate_export_header(mylib
    EXPORT_FILE_NAME ${CMAKE_CURRENT_BINARY_DIR}/include/mylib_export.h
    EXPORT_MACRO_NAME MYLIB_API
    NO_EXPORT_MACRO_NAME MYLIB_HIDDEN
    STATIC_DEFINE MYLIB_STATIC
)

# Set hidden visibility by default (GCC/Clang)
set_target_properties(mylib PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN YES
    VERSION ${PROJECT_VERSION}
    SOVERSION ${PROJECT_VERSION_MAJOR}
)

target_include_directories(mylib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# Support both shared and static builds
# When built as STATIC, MYLIB_STATIC is defined and macros expand to nothing

```

```cpp

// === Generated mylib_export.h looks like: ===
// #ifdef MYLIB_STATIC
//   #define MYLIB_API
//   #define MYLIB_HIDDEN
// #else
//   #ifdef mylib_EXPORTS
//     #define MYLIB_API __declspec(dllexport)  // or __attribute__((visibility("default")))
//   #else
//     #define MYLIB_API __declspec(dllimport)
//   #endif
//   #define MYLIB_HIDDEN
// #endif

```

### Q3: Design ABI-stable library with PIMPL

**Answer:**

```cpp

// === Public header: ABI-stable, never changes layout ===
// include/mylib/engine.h
#include "mylib_export.h"
#include <memory>
#include <string>

namespace mylib {

class MYLIB_API Engine {
public:
    Engine();
    ~Engine();  // Must be in .cpp where Impl is complete

    // Move only (PIMPL)
    Engine(Engine&&) noexcept;
    Engine& operator=(Engine&&) noexcept;

    // Public API: stable across SO versions
    bool initialize(const std::string& config_path);
    void process();
    int status() const;

private:
    class Impl;  // Defined in .cpp only
    std::unique_ptr<Impl> impl_;
};

}  // namespace mylib

// === Implementation: free to change without ABI break ===
// src/engine.cpp
namespace mylib {

class Engine::Impl {
public:
    bool initialize(const std::string& config_path) {
        // Can add/remove/change fields freely
        config_ = load_config(config_path);
        db_ = open_database(config_.db_path);
        return true;
    }
    void process() { /* ... */ }
    int status() const { return status_; }

private:
    // These can change between library versions
    Config config_;
    Database db_;
    int status_ = 0;
    // Adding fields here does NOT break ABI
};

Engine::Engine() : impl_(std::make_unique<Impl>()) {}
Engine::~Engine() = default;
Engine::Engine(Engine&&) noexcept = default;
Engine& Engine::operator=(Engine&&) noexcept = default;

bool Engine::initialize(const std::string& p) {
    return impl_->initialize(p);
}
void Engine::process() { impl_->process(); }
int Engine::status() const { return impl_->status(); }

}  // namespace mylib

```

---

## Notes

- **Always use `-fvisibility=hidden`** (GCC/Clang) — export only what you intend
- CMake's `GenerateExportHeader` handles cross-platform macros automatically
- **PIMPL idiom** is essential for ABI stability: changing Impl doesn't change header layout
- Exported classes should not have inline functions that use non-exported types
- Template classes cannot be exported — they must be in headers (or explicitly instantiated)
- Use `nm -D libmylib.so` or `dumpbin /exports mylib.dll` to inspect exported symbols
- Version your SO: `libmylib.so.1.0.0` with `SOVERSION 1` for compatible updates
