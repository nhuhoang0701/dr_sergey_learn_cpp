# Use Ninja Multi-Config for single-configure multi-config builds

**Category:** Build Systems & CI  
**Item:** #667  
**Reference:** <https://cmake.org/cmake/help/latest/generator/Ninja%20Multi-Config.html>  

---

## Topic Overview

The **Ninja Multi-Config** generator lets you configure CMake **once** and build **multiple configurations** (Debug, Release, RelWithDebInfo) from the same build directory — no reconfigure needed. This saves time in development and CI pipelines.

### Single-Config vs Multi-Config Generators

| Feature | Single-Config (`Ninja`, `Makefiles`) | Multi-Config (`Ninja Multi-Config`, `Visual Studio`) |
| --- | --- | --- |
| CMAKE_BUILD_TYPE | Set at configure time | Set at **build** time via `--config` |
| Configurations | One per build dir | Multiple in same build dir |
| Switching configs | Must reconfigure | Just rebuild with `--config X` |
| Build dir structure | `build/` | `build/Debug/`, `build/Release/` |
| Configure count | Once per config | Once total |

### Workflow Comparison

```cpp

Single-Config (Ninja):
  cmake -B build-debug -G Ninja -DCMAKE_BUILD_TYPE=Debug
  cmake -B build-release -G Ninja -DCMAKE_BUILD_TYPE=Release
  → Two configures, two build dirs

Multi-Config (Ninja Multi-Config):
  cmake -B build -G "Ninja Multi-Config"
  cmake --build build --config Debug
  cmake --build build --config Release
  → One configure, one build dir, multiple configs

```

---

## Self-Assessment

### Q1: Configure with -G 'Ninja Multi-Config' and build both Debug and Release without reconfiguring

**Answer:**

```bash

# ═══════════ Step 1: Configure once ═══════════
cmake -B build -G "Ninja Multi-Config" \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_CONFIGURATION_TYPES="Debug;Release;RelWithDebInfo" \
    -DCMAKE_DEFAULT_BUILD_TYPE=Release
# Output:
# -- The CXX compiler identification is GNU 13.2.0
# -- Configuring done (0.3s)
# -- Generating done (0.1s)
# -- Build files: build/build-Debug.ninja, build/build-Release.ninja

# ═══════════ Step 2: Build Debug ═══════════
cmake --build build --config Debug
# Compiles with -g -O0 flags
# Output: build/Debug/myapp

# ═══════════ Step 3: Build Release (NO reconfigure!) ═══════════
cmake --build build --config Release
# Compiles with -O2 -DNDEBUG flags
# Output: build/Release/myapp

# ═══════════ Step 4: Build RelWithDebInfo ═══════════
cmake --build build --config RelWithDebInfo
# Output: build/RelWithDebInfo/myapp

# ═══════════ Test each config ═══════════
ctest --test-dir build --build-config Debug
ctest --test-dir build --build-config Release

# ═══════════ Install a specific config ═══════════
cmake --install build --config Release --prefix /usr/local

```

```cpp

Build directory structure:
build/
├── build.ninja              ← Main ninja file
├── build-Debug.ninja        ← Debug-specific rules
├── build-Release.ninja      ← Release-specific rules
├── Debug/
│   ├── myapp                ← Debug binary
│   └── libmylib.a
├── Release/
│   ├── myapp                ← Release binary
│   └── libmylib.a
└── CMakeFiles/              ← Shared CMake metadata

```

### Q2: Explain how multi-config generators differ from single-config generators in CMake

**Answer:**

```cmake

# ═══════════ Single-Config Generator (Ninja, Unix Makefiles) ═══════════
# CMAKE_BUILD_TYPE is a CACHE variable set at configure time
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
# Build type is FIXED — changing it requires reconfigure:
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Debug  # Full reconfigure!

# In CMakeLists.txt, CMAKE_BUILD_TYPE is available:
message(STATUS "Build type: ${CMAKE_BUILD_TYPE}")  # "Release"

# Generator expressions work but have only one active config:
target_compile_definitions(app PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>      # Only one triggers
    $<$<CONFIG:Release>:RELEASE_MODE>
)

# ═══════════ Multi-Config Generator (Ninja Multi-Config, VS, Xcode) ═══════════
# CMAKE_BUILD_TYPE is IGNORED — do NOT set it
cmake -B build -G "Ninja Multi-Config"
# ⚠️ -DCMAKE_BUILD_TYPE=Release has NO effect here

# Use CMAKE_CONFIGURATION_TYPES instead:
cmake -B build -G "Ninja Multi-Config" \
    -DCMAKE_CONFIGURATION_TYPES="Debug;Release;RelWithDebInfo;MinSizeRel"

# Config selected at BUILD time:
cmake --build build --config Debug
cmake --build build --config Release

# In CMakeLists.txt, CMAKE_BUILD_TYPE is EMPTY for multi-config
# Use generator expressions for config-dependent logic:
target_compile_definitions(app PRIVATE
    $<$<CONFIG:Debug>:DEBUG_MODE>
    $<$<CONFIG:Release>:RELEASE_MODE>
)
# Both definitions are generated — the active one is selected at build time

# ═══════════ Writing CMakeLists.txt that works with both ═══════════
# Correct:
target_compile_options(app PRIVATE
    $<$<CONFIG:Debug>:-g -O0 -fsanitize=address>
    $<$<CONFIG:Release>:-O2 -DNDEBUG>
)

# Incorrect (only works with single-config):
if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    target_compile_options(app PRIVATE -g -O0)  # ⚠️ Ignored by multi-config!
endif()

```

### Q3: Use --config Release in cmake --build to select the configuration at build time

**Answer:**

```bash

# ═══════════ Configure once ═══════════
cmake -B build -G "Ninja Multi-Config" \
    -DCMAKE_CXX_STANDARD=20 \
    -DCMAKE_DEFAULT_BUILD_TYPE=Debug

# ═══════════ Build with --config ═══════════
# Default config (Debug, since CMAKE_DEFAULT_BUILD_TYPE=Debug):
cmake --build build
# → Builds Debug

# Explicit config — no reconfigure needed:
cmake --build build --config Release
# → Builds Release, output in build/Release/

cmake --build build --config Debug
# → Builds Debug, output in build/Debug/

# ═══════════ Parallel builds of different configs ═══════════
# Since each config has separate output dirs, you can even build in parallel:
cmake --build build --config Debug   &
cmake --build build --config Release &
wait
# Both build simultaneously

# ═══════════ CI pipeline example ═══════════
# .github/workflows/ci.yml
# jobs:
#   build:
#     steps:
#       - run: cmake -B build -G "Ninja Multi-Config"
#       - run: cmake --build build --config Debug
#       - run: ctest --test-dir build --build-config Debug
#       - run: cmake --build build --config Release
#       - run: ctest --test-dir build --build-config Release
#       # One configure, two build+test cycles — faster than two full pipelines

# ═══════════ With CMakePresets.json ═══════════
# {
#   "configurePresets": [{
#     "name": "multi",
#     "generator": "Ninja Multi-Config",
#     "binaryDir": "${sourceDir}/build",
#     "cacheVariables": {
#       "CMAKE_CONFIGURATION_TYPES": "Debug;Release"
#     }
#   }],
#   "buildPresets": [
#     { "name": "debug",   "configurePreset": "multi", "configuration": "Debug" },
#     { "name": "release", "configurePreset": "multi", "configuration": "Release" }
#   ]
# }
#
# cmake --preset multi
# cmake --build --preset debug
# cmake --build --preset release

```

---

## Notes

- **`CMAKE_DEFAULT_BUILD_TYPE`** sets which config `cmake --build build` uses when `--config` is omitted
- **`CMAKE_CROSS_CONFIGS`** (advanced): allows building targets from one config using another's settings (e.g., Debug executable with Release libraries)
- **IDE support:** VS Code CMake Tools, CLion, and Visual Studio all understand multi-config generators
- **Ninja Multi-Config requires CMake 3.17+** and Ninja 1.10+
