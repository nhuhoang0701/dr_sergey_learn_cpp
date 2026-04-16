# Structure a C++ project with clean directory layout and CMake

**Category:** Project Architecture

---

## Topic Overview

A well-structured C++ project separates concerns into clear directories, uses modern CMake idioms, and scales from small libraries to large applications. The directory layout communicates architecture and makes onboarding easy.

### Recommended Directory Structure

```cpp

my_project/
├── CMakeLists.txt              # Root build configuration
├── cmake/                      # CMake modules and toolchains
│   ├── CompilerWarnings.cmake
│   └── arm-toolchain.cmake
├── include/                    # Public headers (installed)
│   └── myproject/
│       ├── core.h
│       └── utils.h
├── src/                        # Implementation files
│   ├── core.cpp
│   ├── utils.cpp
│   └── internal/               # Private headers
│       └── detail.h
├── apps/                       # Executables
│   └── main.cpp
├── tests/                      # Test files
│   ├── CMakeLists.txt
│   ├── test_core.cpp
│   └── test_utils.cpp
├── benchmarks/                 # Performance benchmarks
│   └── bench_core.cpp
├── docs/                       # Documentation
├── third_party/                # Vendored dependencies
├── .clang-tidy
├── .clang-format
└── vcpkg.json                  # Package dependencies

```

---

## Self-Assessment

### Q1: Write a modern CMakeLists.txt for a library + application project

**Answer:**

```cmake

# === Root CMakeLists.txt ===
cmake_minimum_required(VERSION 3.20)
project(myproject
    VERSION 1.2.3
    DESCRIPTION "My awesome C++ project"
    LANGUAGES CXX
)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)

# Generate compile_commands.json for IDE/tools
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Options
option(BUILD_TESTING "Build tests" ON)
option(BUILD_BENCHMARKS "Build benchmarks" OFF)
option(BUILD_DOCS "Build documentation" OFF)

# Include custom CMake modules
list(APPEND CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake)
include(CompilerWarnings)

# === Library target ===
add_library(myproject_lib
    src/core.cpp
    src/utils.cpp
)
target_include_directories(myproject_lib
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
    PRIVATE
        ${CMAKE_SOURCE_DIR}/src/internal
)
add_library(myproject::lib ALIAS myproject_lib)
set_compiler_warnings(myproject_lib)

# === Application target ===
add_executable(myproject_app apps/main.cpp)
target_link_libraries(myproject_app PRIVATE myproject::lib)

# === Tests ===
if(BUILD_TESTING)
    enable_testing()
    add_subdirectory(tests)
endif()

# === Benchmarks ===
if(BUILD_BENCHMARKS)
    add_subdirectory(benchmarks)
endif()

```

```cmake

# === cmake/CompilerWarnings.cmake ===
function(set_compiler_warnings target)
    target_compile_options(${target} PRIVATE
        $<$<CXX_COMPILER_ID:GNU,Clang>:
            -Wall -Wextra -Wpedantic
            -Wshadow -Wnon-virtual-dtor
            -Wconversion -Wsign-conversion
            -Woverloaded-virtual
        >
        $<$<CXX_COMPILER_ID:MSVC>:
            /W4 /permissive-
        >
    )
endfunction()

```

```cmake

# === tests/CMakeLists.txt ===
include(FetchContent)
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG v1.14.0
)
set(INSTALL_GTEST OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(googletest)

add_executable(myproject_tests
    test_core.cpp
    test_utils.cpp
)
target_link_libraries(myproject_tests
    PRIVATE myproject::lib GTest::gtest_main GTest::gmock
)
add_test(NAME myproject_tests COMMAND myproject_tests)

```

### Q2: Configure installation and find_package support

**Answer:**

```cmake

# === Install rules (append to root CMakeLists.txt) ===
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

# Install library
install(TARGETS myproject_lib
    EXPORT myprojectTargets
    LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
    ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
    RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
)

# Install headers
install(DIRECTORY include/myproject
    DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
)

# Install CMake config for find_package
install(EXPORT myprojectTargets
    FILE myprojectTargets.cmake
    NAMESPACE myproject::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
)

configure_package_config_file(
    cmake/myprojectConfig.cmake.in
    ${CMAKE_BINARY_DIR}/myprojectConfig.cmake
    INSTALL_DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
)

write_basic_package_version_file(
    ${CMAKE_BINARY_DIR}/myprojectConfigVersion.cmake
    VERSION ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion
)

install(FILES
    ${CMAKE_BINARY_DIR}/myprojectConfig.cmake
    ${CMAKE_BINARY_DIR}/myprojectConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/myproject
)

```

```cmake

# === cmake/myprojectConfig.cmake.in ===
@PACKAGE_INIT@
include("${CMAKE_CURRENT_LIST_DIR}/myprojectTargets.cmake")
check_required_components(myproject)

```

```cmake

# Consumers can now use:
# find_package(myproject 1.2 REQUIRED)
# target_link_libraries(myapp PRIVATE myproject::lib)

```

### Q3: CMake presets for different build configurations

**Answer:**

```json

{
    "version": 6,
    "cmakeMinimumRequired": {"major": 3, "minor": 25, "patch": 0},
    "configurePresets": [
        {
            "name": "dev",
            "displayName": "Development",
            "binaryDir": "${sourceDir}/build/dev",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "BUILD_TESTING": "ON",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON"
            }
        },
        {
            "name": "release",
            "displayName": "Release",
            "binaryDir": "${sourceDir}/build/release",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "BUILD_TESTING": "OFF"
            }
        },
        {
            "name": "ci",
            "displayName": "CI Build",
            "binaryDir": "${sourceDir}/build/ci",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "BUILD_TESTING": "ON",
                "BUILD_BENCHMARKS": "ON"
            }
        }
    ],
    "buildPresets": [
        {"name": "dev", "configurePreset": "dev"},
        {"name": "release", "configurePreset": "release"},
        {"name": "ci", "configurePreset": "ci"}
    ],
    "testPresets": [
        {
            "name": "dev",
            "configurePreset": "dev",
            "output": {"outputOnFailure": true}
        }
    ]
}

```

```bash

# Usage:
cmake --preset dev
cmake --build --preset dev
ctest --preset dev

```

---

## Notes

- **Public headers go in `include/projectname/`** — the subdirectory prevents include collisions
- Use `$<BUILD_INTERFACE:...>` and `$<INSTALL_INTERFACE:...>` for correct include paths in both local and installed usage
- Always use `ALIAS` targets (`myproject::lib`) — they fail loudly if the target doesn't exist
- `cmake/` directory holds toolchains, find modules, and helper functions
- CMakePresets.json replaces the need for wrapper scripts — `cmake --preset dev` is all you need
- `CMAKE_EXPORT_COMPILE_COMMANDS=ON` is essential for clang-tidy and IDE support
- Separate `apps/` from `src/` to keep library code reusable without the main() entry point
