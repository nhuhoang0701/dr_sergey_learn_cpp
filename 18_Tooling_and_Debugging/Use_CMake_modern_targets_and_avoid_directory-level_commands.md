# Use CMake modern targets and avoid directory-level commands

**Category:** Tooling & Debugging  
**Item:** #146  
**Reference:** <https://cmake.org/cmake/help/latest/>  

---

## Topic Overview

Modern CMake (3.0+) is **target-based**: properties are attached to targets, not directories. This ensures correct dependency propagation.

```cpp

OLD (directory-level):           MODERN (target-level):
  include_directories(...)         target_include_directories(lib PUBLIC ...)
  add_definitions(...)             target_compile_definitions(lib PRIVATE ...)
  add_compile_options(...)         target_compile_options(lib PRIVATE ...)
  link_libraries(...)              target_link_libraries(app PRIVATE lib)

  ✗ Affects ALL targets             ✓ Only affects specified target
  ✗ No dependency propagation       ✓ PUBLIC/PRIVATE/INTERFACE propagation

```

---

## Self-Assessment

### Q1: Target-based CMakeLists.txt with PRIVATE/PUBLIC/INTERFACE

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(ModernCMakeDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# === Library target ===
add_library(mathlib src/math.cpp)

# PUBLIC: consumers also need these
target_include_directories(mathlib PUBLIC
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
)

# PRIVATE: only mathlib needs this internally
target_compile_definitions(mathlib PRIVATE MATHLIB_BUILDING)

# PUBLIC: consumers inherit this flag
target_compile_definitions(mathlib PUBLIC MATHLIB_VERSION=2)

# === Another library that depends on mathlib ===
add_library(physics src/physics.cpp)
target_include_directories(physics PUBLIC include)
target_link_libraries(physics PUBLIC mathlib)  # physics users also get mathlib

# === Application target ===
add_executable(app src/main.cpp)

# PRIVATE: only app links these, not propagated
target_link_libraries(app PRIVATE physics)

# PRIVATE: only app sees these options
target_compile_options(app PRIVATE -Wall -Wextra -Wpedantic)

# === INTERFACE library (header-only) ===
add_library(concepts_lib INTERFACE)
target_include_directories(concepts_lib INTERFACE include/concepts)
target_compile_features(concepts_lib INTERFACE cxx_std_20)

```

**Propagation rules:**

| Keyword | Compiling target | Consumers of target |
| --- | --- | --- |
| PRIVATE | Yes | No |
| PUBLIC | Yes | Yes |
| INTERFACE | No | Yes |

### Q2: Why `target_compile_options()` is better than `add_compile_options()`

```cmake

# BAD: directory-level — affects EVERY target in this directory
add_compile_options(-Wall -Wextra -Werror)
# Problem: third-party subdirectories also get -Werror
# add_subdirectory(third_party/noisy_lib)  # fails to compile

# GOOD: target-level — only your targets get strict warnings
target_compile_options(mylib PRIVATE -Wall -Wextra -Werror)
target_compile_options(myapp PRIVATE -Wall -Wextra -Werror)
# third_party/noisy_lib is unaffected

# For libraries: PUBLIC flags propagate to consumers
# BAD: forces YOUR warnings onto all consumers
target_compile_options(mylib PUBLIC -Werror)  # Don't do this!

# GOOD: keep strictness PRIVATE
target_compile_options(mylib PRIVATE -Werror)

```

| Approach | Scope | Third-party safe? | Recommendation |
| --- | --- | --- | --- |
| `add_compile_options()` | All targets in directory | No | Avoid |
| `target_compile_options(PRIVATE)` | Single target | Yes | Use this |
| `target_compile_options(PUBLIC)` | Target + consumers | Depends | Rarely needed |

### Q3: Proper `install()` rules and export sets

```cmake

# CMakeLists.txt (continued)
include(GNUInstallDirs)

# Install the library and headers
install(TARGETS mathlib physics
    EXPORT MyProjectTargets   # Create an export set
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

install(DIRECTORY include/
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Generate and install the config file for find_package()
install(EXPORT MyProjectTargets
    FILE MyProjectTargets.cmake
    NAMESPACE MyProject::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)

# Create MyProjectConfig.cmake
include(CMakePackageConfigHelpers)
configure_package_config_file(
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/MyProjectConfig.cmake.in
    ${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)

install(FILES
    ${CMAKE_CURRENT_BINARY_DIR}/MyProjectConfig.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MyProject
)

```

```cmake

# cmake/MyProjectConfig.cmake.in
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/MyProjectTargets.cmake")
check_required_components(MyProject)

```

Downstream consumer:

```cmake

# Consumer's CMakeLists.txt
find_package(MyProject REQUIRED)
target_link_libraries(consumer_app PRIVATE MyProject::mathlib)
# Automatically gets: include dirs, compile defs, link flags

```

---

## Notes

- Never use `include_directories()`, `link_libraries()`, `add_definitions()` in modern CMake.
- Use `target_compile_features(lib PUBLIC cxx_std_20)` instead of `set(CMAKE_CXX_STANDARD 20)`.
- Generator expressions (`$<BUILD_INTERFACE:...>`) handle build vs install paths.
- `INTERFACE` libraries are perfect for header-only libraries.
- Use `cmake --install build --prefix /install/path` to test install rules locally.
