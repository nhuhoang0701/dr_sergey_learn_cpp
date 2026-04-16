# Use CMakePresets.json for reproducible, shareable build configurations

**Category:** Build Systems & CI  
**Item:** #659  
**Reference:** <https://cmake.org/cmake/help/latest/manual/cmake-presets.7.html>  

---

## Topic Overview

This second file on CMakePresets focuses on the **practical workflow**: using `cmake --preset=release` to configure without remembering flags, and how `CMakeUserPresets.json` lets each developer customize without touching shared configuration.

### Preset Workflow

```bash

# Before presets (error-prone):
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_CXX_STANDARD=20 ...

# With presets (one command, reproducible):
cmake --preset release
cmake --build --preset release
ctest --preset release

```

---

## Self-Assessment

### Q1: Write a CMakePresets.json with configure, build, and test presets for Debug and Release

**Answer:**

```json

{
    "version": 6,
    "configurePresets": [
        {
            "name": "ci-base",
            "hidden": true,
            "generator": "Ninja",
            "binaryDir": "${sourceDir}/build/${presetName}",
            "cacheVariables": {
                "CMAKE_CXX_STANDARD": "20",
                "CMAKE_EXPORT_COMPILE_COMMANDS": "ON",
                "BUILD_TESTING": "ON"
            }
        },
        {
            "name": "debug",
            "displayName": "Debug Build",
            "inherits": "ci-base",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Debug",
                "CMAKE_CXX_FLAGS": "-g -O0 -Wall -Wextra -Wpedantic -Werror"
            }
        },
        {
            "name": "release",
            "displayName": "Release Build",
            "inherits": "ci-base",
            "cacheVariables": {
                "CMAKE_BUILD_TYPE": "Release",
                "CMAKE_CXX_FLAGS": "-O2 -Wall -Wextra -DNDEBUG"
            }
        },
        {
            "name": "coverage",
            "displayName": "Coverage Build",
            "inherits": "debug",
            "cacheVariables": {
                "CMAKE_CXX_FLAGS": "-g -O0 --coverage -fprofile-arcs -ftest-coverage"
            }
        }
    ],
    "buildPresets": [
        {
            "name": "debug",
            "configurePreset": "debug"
        },
        {
            "name": "release",
            "configurePreset": "release"
        },
        {
            "name": "coverage",
            "configurePreset": "coverage"
        }
    ],
    "testPresets": [
        {
            "name": "debug",
            "configurePreset": "debug",
            "output": { "outputOnFailure": true },
            "execution": { "timeout": 120 }
        },
        {
            "name": "release",
            "configurePreset": "release",
            "output": { "outputOnFailure": true }
        }
    ]
}

```

```bash

# Full workflow:
cmake --preset debug && cmake --build --preset debug && ctest --preset debug
cmake --preset release && cmake --build --preset release && ctest --preset release

```

### Q2: Use cmake --preset=release to configure and build without manually specifying -D flags

**Answer:**

```bash

# ═══════════ Without presets (manual flags) ═══════════
cmake -B build-release \
    -G Ninja \
    -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_CXX_STANDARD_REQUIRED=ON \
    -DCMAKE_EXPORT_COMPILE_COMMANDS=ON \
    -DBUILD_TESTING=ON \
    -DCMAKE_CXX_FLAGS="-O2 -Wall -Wextra -DNDEBUG"
cmake --build build-release --parallel $(nproc)
cd build-release && ctest --output-on-failure

# Problems:
# - Developer might forget -DBUILD_TESTING=ON
# - Different devs use different generators (Make vs Ninja)
# - Flag typo? Build silently misconfigured

# ═══════════ With presets (reproducible) ═══════════
cmake --preset release
# Output:
# Preset CMake variables:
#   CMAKE_BUILD_TYPE="Release"
#   CMAKE_CXX_FLAGS="-O2 -Wall -Wextra -DNDEBUG"
#   CMAKE_CXX_STANDARD="20"
#   CMAKE_EXPORT_COMPILE_COMMANDS="ON"
#   BUILD_TESTING="ON"
# -- Configuring done
# -- Generating done
# -- Build files have been written to: /project/build/release

cmake --build --preset release
ctest --preset release

# Every developer, CI runner, and IDE gets IDENTICAL configuration
# No need to read a wiki page for "how to build this project"

# ═══════════ What cmake --preset does internally ═══════════
# It reads CMakePresets.json, finds the "release" preset, and runs:
#   cmake -B build/release -G Ninja \
#     -DCMAKE_BUILD_TYPE=Release \
#     ... (all cacheVariables from the preset)

```

### Q3: Explain how CMakeUserPresets.json (gitignored) extend shared presets for per-developer overrides

**Answer:**

```json

// CMakeUserPresets.json — ALWAYS in .gitignore
{
    "version": 6,
    "configurePresets": [
        {
            "name": "my-dev",
            "inherits": "debug",
            "displayName": "Alice's Dev Build",
            "cacheVariables": {
                "CMAKE_INSTALL_PREFIX": "/home/alice/opt/myapp",
                "CMAKE_PREFIX_PATH": "/home/alice/vcpkg/installed/x64-linux",
                "ENABLE_EXPENSIVE_TESTS": "OFF"
            },
            "environment": {
                "CXX": "/usr/bin/clang++-17"
            }
        },
        {
            "name": "my-arm",
            "inherits": "release",
            "toolchainFile": "/home/alice/toolchains/aarch64.cmake",
            "binaryDir": "${sourceDir}/build/arm64"
        }
    ],
    "buildPresets": [
        {
            "name": "my-dev",
            "configurePreset": "my-dev",
            "jobs": 16
        }
    ]
}

```

```bash

# Alice uses her presets:
cmake --preset my-dev
cmake --build --preset my-dev

# Bob doesn't have CMakeUserPresets.json — uses shared presets:
cmake --preset debug

# What Alice's .gitignore contains:
# CMakeUserPresets.json
# build/

# Inheritance rules:
# CMakeUserPresets.json CAN inherit from CMakePresets.json presets ✅
# CMakePresets.json CANNOT inherit from CMakeUserPresets.json ✗
# → Shared config never depends on personal config

```

**Use cases for user presets:**

- Custom install path (different on each machine)
- Custom vcpkg/Conan path
- Preferred compiler (Clang vs GCC)
- Cross-compilation toolchain files
- Development-only flags (disable slow tests, enable extra logging)

---

## Notes

- **IDE support:** VS Code (CMake Tools extension), CLion, and Visual Studio 2022 all read `CMakePresets.json` and show presets in their UI
- **`${presetName}`** variable in `binaryDir` creates per-preset build directories automatically
- **Vendor fields:** add `"vendor": {"my-company": {...}}` for custom metadata that CMake ignores
- **Workflow presets** (version 6): chain configure → build → test into a single `cmake --workflow --preset ci` command
