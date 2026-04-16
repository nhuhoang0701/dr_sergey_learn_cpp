# Use AI assistants for CMake and build system configuration

**Category:** AI-Assisted C++ Development

---

## Topic Overview

CMake configuration is one of the most frustrating parts of C++ development. AI assistants can generate **CMakeLists.txt** from scratch, debug build issues, add dependencies correctly, configure cross-compilation, and translate between build systems (Makefile → CMake, Bazel → CMake). AI is especially valuable because CMake's documentation is dense and its patterns are non-obvious.

### AI CMake Assistance Areas

| Task | AI Effectiveness | Example Prompt |
| --- | --- | --- |
| Generate CMakeLists.txt | Excellent | "Create CMake for this project structure" |
| Add vcpkg/Conan dependency | Excellent | "Add gRPC with vcpkg to this CMake" |
| Fix build errors | Good | "Why does this CMake error occur?" |
| Cross-compilation toolchain | Good | "CMake toolchain for ARM Cortex-M4" |
| CTest integration | Excellent | "Add GTest with CTest to this project" |
| CPack packaging | Good | "Generate DEB/RPM packages" |
| CMake presets | Good | "Create presets for debug/release/CI" |

---

## Self-Assessment

### Q1: Generate project CMake from scratch

**Answer:**

```cpp

=== PROMPT ===

"Generate a modern CMakeLists.txt (CMake 3.25+) for this
project structure. Use target-based commands only (no
global include_directories or link_libraries).
Dependencies: fmt, spdlog, GTest, protobuf.
Package manager: vcpkg.

my_project/
  src/
    main.cpp
    server/
      server.cpp
      server.hpp
    proto/
      messages.proto
  include/
    my_project/
      config.hpp
      types.hpp
  tests/
    server_test.cpp
    integration_test.cpp
  CMakeLists.txt"

```

```cmake

# === AI-generated CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(my_project
    VERSION 1.0.0
    LANGUAGES CXX
)

# ── Standards ──
set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# ── Dependencies (vcpkg) ──
find_package(fmt CONFIG REQUIRED)
find_package(spdlog CONFIG REQUIRED)
find_package(Protobuf CONFIG REQUIRED)
find_package(GTest CONFIG REQUIRED)

# ── Protobuf generation ──
set(PROTO_FILES src/proto/messages.proto)
protobuf_generate_cpp(PROTO_SRCS PROTO_HDRS ${PROTO_FILES})

# ── Main library (shared between app and tests) ──
add_library(my_project_lib
    src/server/server.cpp
    ${PROTO_SRCS}
)
target_include_directories(my_project_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_CURRENT_SOURCE_DIR}/src
        ${CMAKE_CURRENT_BINARY_DIR}  # For generated proto headers
)
target_link_libraries(my_project_lib
    PUBLIC
        fmt::fmt
        spdlog::spdlog
        protobuf::libprotobuf
)
target_compile_options(my_project_lib PRIVATE
    $<$<CXX_COMPILER_ID:GNU,Clang>:-Wall -Wextra -Wpedantic>
    $<$<CXX_COMPILER_ID:MSVC>:/W4>
)

# ── Executable ──
add_executable(my_project src/main.cpp)
target_link_libraries(my_project PRIVATE my_project_lib)

# ── Tests ──
enable_testing()
add_executable(my_project_tests
    tests/server_test.cpp
    tests/integration_test.cpp
)
target_link_libraries(my_project_tests PRIVATE
    my_project_lib
    GTest::gtest_main
)

include(GoogleTest)
gtest_discover_tests(my_project_tests)

```

### Q2: Debug CMake build errors

**Answer:**

```cpp

=== PROMPT ===

"Explain this CMake error and show the fix:

CMake Error at CMakeLists.txt:18 (find_package):
  By not providing 'FindBoost.cmake' in
  CMAKE_MODULE_PATH this project has asked CMake to
  find a package configuration file provided by 'Boost',
  but CMake did not find one.

  Could not find a package configuration file provided by
  'Boost' (requested version 1.82) with any of the
  following names:
    BoostConfig.cmake
    boost-config.cmake

I installed Boost via vcpkg. Using vcpkg toolchain file."


=== AI DIAGNOSIS ===

```

```cmake

# Problem: vcpkg's Boost doesn't always provide config-mode files
# The error means CMake can't find BoostConfig.cmake

# Fix 1: Use MODULE mode (FindBoost.cmake ships with CMake)
find_package(Boost 1.82 REQUIRED
    COMPONENTS filesystem system
    # No CONFIG keyword - uses FindBoost.cmake
)

# Fix 2: If using vcpkg, ensure toolchain is set BEFORE project()
cmake_minimum_required(VERSION 3.25)

# This MUST be before project():
if(DEFINED ENV{VCPKG_ROOT})
    set(CMAKE_TOOLCHAIN_FILE
        "$ENV{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake"
        CACHE STRING "")
endif()

project(my_project LANGUAGES CXX)

# Fix 3: Use individual Boost components via vcpkg
find_package(boost_filesystem CONFIG REQUIRED)
find_package(boost_system CONFIG REQUIRED)
target_link_libraries(my_target PRIVATE
    Boost::filesystem
    Boost::system
)

```

### Q3: Generate CMake presets and cross-compilation

**Answer:**

```cpp

=== PROMPT ===

"Generate CMakePresets.json with:

- Debug and Release configurations
- GCC and Clang compiler options
- Sanitizer builds (ASan, TSan, UBSan)
- CI build preset (Release + LTO + static analysis)
- vcpkg integration"

```

```json

{
    "version": 6,
    "cmakeMinimumRequired": { "major": 3, "minor": 25, "patch": 0 },
    "configurePresets": [
        {
            "name": "default",
            "hidden": true,
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_TOOLCHAIN_FILE": {
                    "value": "$env{VCPKG_ROOT}/scripts/buildsystems/vcpkg.cmake",
                    "type": "FILEPATH"
                },
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "debug",
            "inherits": "default",
            "displayName": "Debug",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug"
            }
        },
        {
            "name": "release",
            "inherits": "default",
            "displayName": "Release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_INTERPROCEDURAL_OPTIMIZATION": "ON"
            }
        },
        {
            "name": "asan",
            "inherits": "debug",
            "displayName": "Debug + ASan",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer"
            }
        },
        {
            "name": "tsan",
            "inherits": "debug",
            "displayName": "Debug + TSan",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-fsanitize=thread -fno-omit-frame-pointer"
            }
        },
        {
            "name": "ci",
            "inherits": "release",
            "displayName": "CI Build",
            "cacheVariables": {
                "CMAKE_CXX_CLANG_TIDY": "clang-tidy;--warnings-as-errors=*"
            }
        }
    ],
    "buildPresets": [
        { "name": "debug", "configurePreset": "debug" },
        { "name": "release", "configurePreset": "release" },
        { "name": "ci", "configurePreset": "ci", "jobs": 4 }
    ],
    "testPresets": [
        {
            "name": "debug",
            "configurePreset": "debug",
            "output": { "outputOnFailure": true }
        }
    ]
}

```

---

## Notes

- Always use **target-based CMake** (`target_link_libraries`, `target_include_directories`), never global commands
- AI is excellent at **translating between build systems** — "convert this Makefile to CMake"
- For **vcpkg**, the toolchain file must be set **before** `project()` — common AI-generated bug
- Ask AI to generate **cmake-presets.json** instead of bash scripts for build configurations
- AI can generate **FetchContent** blocks for header-only libraries without a package manager
- For cross-compilation, provide the **target triple** and **sysroot path** in the prompt
- CMake's `EXPORT_COMPILE_COMMANDS` is essential for IDE integration — always enable it
