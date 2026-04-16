# Use CMake's FetchContent for dependency management without a package manager

**Category:** Build Systems & CI  
**Item:** #664  
**Reference:** <https://cmake.org/cmake/help/latest/module/FetchContent.html>  

---

## Topic Overview

`FetchContent` downloads and builds dependencies at **configure time** directly from Git repositories, URLs, or archives — no package manager (Conan, vcpkg) needed. It's the simplest way to add third-party libraries to a CMake project.

### FetchContent vs find_package

| Aspect | `FetchContent` | `find_package` |
| --- | --- | --- |
| Source | Downloads & builds from source | Uses pre-installed library |
| Speed | Slower (compiles deps every clean build) | Fast (already compiled) |
| Version control | Pinned via GIT_TAG/URL_HASH | Whatever is installed |
| Offline builds | Needs network (or cached) | Works offline |
| Install pollution | Can pollute install target | No issue |
| Use case | Small-medium deps, CI | System libs, large deps |

### FetchContent Workflow

```bash

cmake -B build
  ├── FetchContent_Declare()  ── records URL/tag
  ├── FetchContent_MakeAvailable()
  │     ├── Downloads source (if not cached)
  │     ├── Adds as subdirectory
  │     └── Targets now available
  └── target_link_libraries(myapp PRIVATE gtest_main)

```

---

## Self-Assessment

### Q1: Use FetchContent_Declare and FetchContent_MakeAvailable to pull googletest from GitHub

**Answer:**

```cmake

cmake_minimum_required(VERSION 3.24)
project(MyProject LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# ═══════════ FetchContent setup ═══════════
include(FetchContent)

# Declare googletest — pin to a specific commit for reproducibility
FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0          # Always pin a tag or commit hash
    GIT_SHALLOW    TRUE             # Don't download full history
    FIND_PACKAGE_ARGS               # Try find_package first (CMake 3.24+)
)

# Declare fmt library
FetchContent_Declare(
    fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
    GIT_SHALLOW    TRUE
)

# Declare from a URL archive (faster than git clone)
FetchContent_Declare(
    nlohmann_json
    URL      https://github.com/nlohmann/json/releases/download/v3.11.3/json.tar.xz
    URL_HASH SHA256=d6c65aca6b1ed68e7a182f4757f0f1c0e0a8c4e09dce5efec41dadb23b71a79b
)

# Download and make available (downloads only on first configure)
FetchContent_MakeAvailable(googletest fmt nlohmann_json)

# ═══════════ Use the fetched libraries ═══════════
add_executable(myapp src/main.cpp)
target_link_libraries(myapp PRIVATE fmt::fmt nlohmann_json::nlohmann_json)

# Tests
enable_testing()
add_executable(myapp_test tests/test_main.cpp)
target_link_libraries(myapp_test PRIVATE GTest::gtest_main fmt::fmt)

include(GoogleTest)
gtest_discover_tests(myapp_test)

```

```bash

# Build and test:
cmake -B build -G Ninja
# -- [fetchcontent] googletest: downloading
# -- [fetchcontent] fmt: downloading
cmake --build build
ctest --test-dir build --output-on-failure
# Second configure reuses cached downloads — no re-download

```

**Key options for FetchContent_Declare:**

- `GIT_TAG` — always pin to a tag or commit SHA (never `main`)
- `GIT_SHALLOW TRUE` — faster downloads
- `URL_HASH SHA256=...` — integrity verification for URL downloads
- `FIND_PACKAGE_ARGS` (CMake 3.24+) — try system-installed version first, fetch only if missing

### Q2: Explain the difference between FetchContent (builds from source) and find_package (uses installed)

**Answer:**

```cmake

# ═══════════ Strategy 1: FetchContent (always builds from source) ═══════════
include(FetchContent)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
)
FetchContent_MakeAvailable(fmt)
# Downloads fmt source, adds as subdirectory, compiles it with your project
# Pros: Always correct version, works on any machine, no pre-install needed
# Cons: Adds to build time, may conflict with other fetched deps

# ═══════════ Strategy 2: find_package (uses pre-installed) ═══════════
find_package(fmt 10.2 REQUIRED)
target_link_libraries(myapp PRIVATE fmt::fmt)
# Looks for FmtConfig.cmake or Findfmt.cmake in CMAKE_PREFIX_PATH
# Pros: Fast (already compiled), works offline
# Cons: Must be installed first (apt, vcpkg, manual build)

# ═══════════ Strategy 3: Hybrid (CMake 3.24+) — best of both ═══════════
include(FetchContent)
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG 10.2.1
    FIND_PACKAGE_ARGS 10.2    # Try find_package(fmt 10.2) first
)
FetchContent_MakeAvailable(fmt)
# If fmt 10.2+ is installed → uses it (fast)
# If not installed → downloads and builds from source (reliable)
# Best for CI: pre-install deps in Docker image for speed,
# but FetchContent handles missing deps gracefully

# ═══════════ When to use which ═══════════
# FetchContent:
#   - Small/medium libraries (googletest, fmt, nlohmann_json)
#   - Projects that must clone-and-build with no prerequisites
#   - CI environments without pre-installed deps
#
# find_package:
#   - Large libraries (Boost, Qt, OpenCV)
#   - System libraries (OpenSSL, zlib)
#   - When build time matters
#
# FIND_PACKAGE_ARGS (hybrid):
#   - Default choice for modern CMake 3.24+

```

### Q3: Show how EXCLUDE_FROM_ALL prevents fetched dependencies from polluting the install target

**Answer:**

```cmake

# ═══════════ Problem: FetchContent installs deps alongside your library ═══════════
include(FetchContent)
FetchContent_Declare(fmt GIT_REPOSITORY ... GIT_TAG 10.2.1)
FetchContent_MakeAvailable(fmt)

# Running 'cmake --install build' installs:
#   /usr/local/lib/libfmt.a         ← Unwanted
#   /usr/local/include/fmt/...      ← Unwanted
#   /usr/local/lib/libmylib.a       ← Your library (wanted)

# ═══════════ Solution 1: EXCLUDE_FROM_ALL (CMake 3.28+) ═══════════
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
    EXCLUDE_FROM_ALL                # <── Prevents install of fmt targets
)
FetchContent_MakeAvailable(fmt)

# Now 'cmake --install build' only installs YOUR targets
# fmt is compiled and linked, but not installed

# ═══════════ Solution 2: Manual EXCLUDE_FROM_ALL (older CMake) ═══════════
FetchContent_Declare(fmt
    GIT_REPOSITORY https://github.com/fmtlib/fmt.git
    GIT_TAG        10.2.1
)
FetchContent_GetProperties(fmt)
if(NOT fmt_POPULATED)
    FetchContent_Populate(fmt)
    add_subdirectory(${fmt_SOURCE_DIR} ${fmt_BINARY_DIR} EXCLUDE_FROM_ALL)
endif()
# Same effect, but works with CMake < 3.28

# ═══════════ Solution 3: Override install() in fetched project ═══════════
# Some projects check for this variable:
set(FMT_INSTALL OFF CACHE BOOL "" FORCE)
FetchContent_MakeAvailable(fmt)

# ═══════════ Verify no pollution ═══════════
# cmake --install build --prefix /tmp/test-install
# ls /tmp/test-install/lib/
#   libmylib.a       ✅ Only your library
# ls /tmp/test-install/include/
#   mylib/           ✅ Only your headers

```

---

## Notes

- **Cache directory:** FetchContent stores downloads in `_deps/` inside the build tree — add to `.gitignore`
- **`FETCHCONTENT_FULLY_DISCONNECTED=ON`** — forces offline builds using previously cached sources
- **`FETCHCONTENT_UPDATES_DISCONNECTED=ON`** — skips update checks for already-downloaded deps
- **Override source:** `FETCHCONTENT_SOURCE_DIR_<NAME>=/path/to/local/clone` — use a local checkout instead of downloading (great for development of a dependency)
