# Structure a monorepo for multiple C++ libraries and applications

**Category:** Project Architecture

---

## Topic Overview

A **monorepo** keeps multiple libraries, applications, and shared tooling in a single repository. For C++ projects this simplifies dependency management, ensures version consistency, and enables atomic cross-project refactoring. The challenge is organizing CMake to handle independent builds, selective compilation, and team ownership.

### Monorepo vs Polyrepo

| Aspect | Monorepo | Polyrepo |
| --- | --- | --- |
| Dependency sync | Always consistent | Version pinning, diamond deps |
| Cross-project refactor | Atomic commit | Multi-repo PR coordination |
| Build time | Potentially large | Per-repo, fast |
| CI/CD | Complex (selective builds) | Simple per-repo |
| Code ownership | Need CODEOWNERS | Natural per-repo |

---

## Self-Assessment

### Q1: Design a monorepo directory layout

**Answer:**

```cpp

project-root/
├─ CMakeLists.txt              # Top-level: orchestrates everything
├─ cmake/
│   ├─ CompilerOptions.cmake    # Shared compiler settings
│   ├─ Coverage.cmake           # Coverage targets
│   └─ Sanitizers.cmake         # ASan/UBSan/TSan settings
├─ libs/                        # Shared libraries
│   ├─ core/
│   │   ├─ CMakeLists.txt
│   │   ├─ include/core/
│   │   ├─ src/
│   │   └─ tests/
│   ├─ networking/
│   │   ├─ CMakeLists.txt
│   │   ├─ include/networking/
│   │   ├─ src/
│   │   └─ tests/
│   └─ serialization/
│       └─ ...
├─ apps/                        # Applications
│   ├─ server/
│   │   ├─ CMakeLists.txt
│   │   ├─ src/
│   │   └─ tests/
│   └─ cli/
│       ├─ CMakeLists.txt
│       └─ src/
├─ tools/                       # Dev tools, code generators
│   ├─ proto_gen/
│   └─ bench/
├─ third_party/                 # Vendored or FetchContent deps
│   ├─ CMakeLists.txt
│   ├─ googletest/
│   └─ fmt/
├─ docs/
├─ .github/CODEOWNERS
└─ CMakePresets.json

```

```cmake

# === Top-level CMakeLists.txt ===
cmake_minimum_required(VERSION 3.25)
project(myproject VERSION 1.0.0)

# Shared settings
include(cmake/CompilerOptions.cmake)

# Options to build subsets
option(BUILD_TESTING "Build tests" ON)
option(BUILD_APPS "Build applications" ON)
option(BUILD_TOOLS "Build tools" OFF)

# Third-party dependencies
add_subdirectory(third_party)

# Libraries (always built)
add_subdirectory(libs/core)
add_subdirectory(libs/networking)
add_subdirectory(libs/serialization)

# Applications (optional)
if(BUILD_APPS)
    add_subdirectory(apps/server)
    add_subdirectory(apps/cli)
endif()

# Tools (optional)
if(BUILD_TOOLS)
    add_subdirectory(tools/proto_gen)
endif()

```

### Q2: Design per-library CMake with proper target export

**Answer:**

```cmake

# === libs/core/CMakeLists.txt ===
add_library(core
    src/logger.cpp
    src/config.cpp
    src/thread_pool.cpp
)
add_library(myproject::core ALIAS core)

target_include_directories(core
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)
target_compile_features(core PUBLIC cxx_std_20)

# === libs/networking/CMakeLists.txt ===
add_library(networking
    src/tcp_server.cpp
    src/http_handler.cpp
)
add_library(myproject::networking ALIAS networking)

target_link_libraries(networking
    PUBLIC myproject::core       # Uses core's public API
    PRIVATE asio                 # Implementation detail
)
target_include_directories(networking
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
        $<INSTALL_INTERFACE:include>
)

# === apps/server/CMakeLists.txt ===
add_executable(server
    src/main.cpp
    src/routes.cpp
)
target_link_libraries(server PRIVATE
    myproject::networking
    myproject::serialization
)

# === Tests co-located with library ===
# libs/core/tests/CMakeLists.txt
if(BUILD_TESTING)
    add_executable(core_tests
        test_logger.cpp
        test_config.cpp
        test_thread_pool.cpp
    )
    target_link_libraries(core_tests PRIVATE
        myproject::core
        GTest::gtest_main
    )
    add_test(NAME core_tests COMMAND core_tests)
endif()

```

### Q3: Selective CI builds for changed components

**Answer:**

```yaml

# === .github/workflows/ci.yml ===
name: CI
on: [push, pull_request]

jobs:
  detect-changes:
    runs-on: ubuntu-latest
    outputs:
      core: ${{ steps.filter.outputs.core }}
      networking: ${{ steps.filter.outputs.networking }}
      server: ${{ steps.filter.outputs.server }}
    steps:

      - uses: actions/checkout@v4
      - uses: dorny/paths-filter@v2

        id: filter
        with:
          filters: |
            core:

              - 'libs/core/**'
              - 'cmake/**'

            networking:

              - 'libs/networking/**'
              - 'libs/core/**'          # networking depends on core

            server:

              - 'apps/server/**'
              - 'libs/**'               # server depends on all libs

  build-core:
    needs: detect-changes
    if: needs.detect-changes.outputs.core == 'true'
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - run: |

          cmake -B build -DBUILD_APPS=OFF -DBUILD_TOOLS=OFF
          cmake --build build
          ctest --test-dir build --output-on-failure

  build-server:
    needs: detect-changes
    if: needs.detect-changes.outputs.server == 'true'
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4
      - run: |

          cmake -B build -DBUILD_TESTING=ON
          cmake --build build --target server
          cmake --build build --target server_tests
          ctest --test-dir build -R server --output-on-failure

```

```cpp

# === .github/CODEOWNERS ===
libs/core/           @team-platform
libs/networking/     @team-networking
libs/serialization/  @team-data
apps/server/         @team-backend
apps/cli/            @team-cli
cmake/               @team-platform

```

---

## Notes

- Use **ALIAS targets** (`myproject::core`) for a clean namespace and instant error detection
- Each library should be independently testable with its own `add_test` targets
- **Selective CI** avoids rebuilding the entire monorepo on every push
- `CODEOWNERS` enforces review by the owning team
- `CMakePresets.json` standardizes build configurations across developers
- Use `FetchContent` or `add_subdirectory(third_party/...)` for external dependencies
- For very large monorepos (100+ libs), consider Bazel or build caching (ccache, sccache)
