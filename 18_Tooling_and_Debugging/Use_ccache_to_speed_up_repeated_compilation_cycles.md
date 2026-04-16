# Use ccache to speed up repeated compilation cycles

**Category:** Tooling & Debugging  
**Item:** #416  
**Reference:** <https://ccache.dev>  

---

## Topic Overview

ccache stores the mapping from `hash(preprocessed source + flags)` → `object file`. When the same compilation is requested again, it returns the cached result in milliseconds.

```cpp

Cache key = hash of:

  1. Preprocessed source code (after #include expansion)
  2. Compiler binary (path + content hash)
  3. Compiler flags (-O2, -std=c++20, etc.)
  4. Target architecture

Cache value = compiled object file (.o)

```

---

## Self-Assessment

### Q1: CMake integration with `CMAKE_CXX_COMPILER_LAUNCHER`

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(FastBuild LANGUAGES CXX)

# Auto-detect ccache
find_program(CCACHE ccache)
if(CCACHE)
    message(STATUS "Using ccache: ${CCACHE}")
    set(CMAKE_CXX_COMPILER_LAUNCHER ${CCACHE})
    set(CMAKE_C_COMPILER_LAUNCHER ${CCACHE})
else()
    message(STATUS "ccache not found; building without cache")
endif()

set(CMAKE_CXX_STANDARD 20)
add_executable(app main.cpp engine.cpp renderer.cpp physics.cpp)

```

```bash

# Build with ccache:
cmake -B build && cmake --build build -j$(nproc)

# Or override from command line:
cmake -B build -DCMAKE_CXX_COMPILER_LAUNCHER=ccache

```

### Q2: Cache hit/miss statistics after a clean + rebuild cycle

```bash

# 1. Clear everything
ccache -C && ccache -z

# 2. First build (all misses)
cmake --build build --clean-first -j$(nproc)
ccache -s

```

Output after first build:

```text

cache directory                     /home/user/.cache/ccache
primary config                      /home/user/.config/ccache/ccache.conf
cache hit (direct)                     0
cache hit (preprocessed)               0
cache miss                            42
called for preprocessing               0
files in cache                        42
cache size                          12.3 MB

```

```bash

# 3. Clean and rebuild (all hits!)
cmake --build build --clean-first -j$(nproc)
ccache -s

```

Output after second build:

```text

cache hit (direct)                    42   # <-- all hits!
cache hit (preprocessed)               0
cache miss                            42   # from first build
files in cache                        42
cache size                          12.3 MB

```

**Hit rate interpretation:**

- `direct` hit: fastest path, hash matches without preprocessing
- `preprocessed` hit: fallback, had to preprocess first then match
- `miss`: new or changed compilation, must actually compile

### Q3: What ccache caches and when it invalidates

ccache caches `preprocessed_source + flags -> object_file`:

```cpp

Source file:  main.cpp
  │
  ├─ #include <vector>    ──┐
  ├─ #include "config.h"  ─┤  All expanded into
  └─ your code            ─┤  one preprocessed blob
                            │
                     hash(blob + flags + compiler)
                            │
                     cache lookup
                     ┌────┴────┐
                   HIT         MISS
                   │             │
               return .o    compile -> store .o

```

**Invalidation triggers:**

| Event | Result | Explanation |
| --- | --- | --- |
| Edit `main.cpp` | Miss | Source content changed |
| Edit `config.h` | Miss | Included header changed |
| Change `-O2` to `-O3` | Miss | Flags are part of hash |
| Upgrade GCC 13 → 14 | Miss | Compiler binary changed |
| Switch branches | Depends | Hit if same code, miss if different |
| `git stash pop` (restore) | **Hit** | Same code as before = same hash |
| Add a comment | **Hit** | Preprocessor removes comments |
| Change `__TIME__` macro | Miss | Preprocessed output differs |

---

## Notes

- ccache default cache size is 5GB; increase with `ccache --set-config=max_size=20G`.
- Use `ccache -p` to see full configuration.
- ccache supports remote caching (Redis, HTTP) for shared CI caches.
- sccache is an alternative with native cloud storage support (S3, GCS, Azure).
- Combine ccache with Ninja for fastest possible builds.
