# Use sanitizer-driven testing as part of the CI pipeline

**Category:** Testing & Verification  
**Item:** #769  
**Standard:** C++17  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Sanitizers are **compiler instrumentation** tools that detect bugs at runtime that normal tests miss: memory errors, undefined behavior, data races, and leaks. Integrating them into CI ensures every commit is automatically checked.

### Sanitizer Summary

| Sanitizer | Flag | Detects | Overhead |
| --- | --- | --- | --- |
| **AddressSanitizer (ASAN)** | `-fsanitize=address` | Buffer overflow, use-after-free, double-free, stack overflow | ~2× slower, 2-3× memory |
| **UndefinedBehaviorSanitizer (UBSAN)** | `-fsanitize=undefined` | Signed overflow, null deref, alignment, shift errors | ~1.2× slower |
| **ThreadSanitizer (TSAN)** | `-fsanitize=thread` | Data races, lock-order inversion | ~5-15× slower, 5-10× memory |
| **MemorySanitizer (MSAN)** | `-fsanitize=memory` | Reads of uninitialized memory | ~3× slower |
| **LeakSanitizer (LSAN)** | `-fsanitize=leak` | Memory leaks (included in ASAN) | Minimal at exit |

### Architecture: Sanitizers in CI

```cpp

┌──────────────────────────────────────────────────┐
│                    CI Pipeline                      │
├─────────────┬───────────────┬────────────────────┤
│  Job 1      │  Job 2        │  Job 3              │
│  Release    │  ASAN+UBSAN   │  TSAN               │
│  -O2        │  -O1 -g       │  -O1 -g             │
│  No sanitize│  -fsanitize=  │  -fsanitize=thread  │
│             │  address,     │                      │
│             │  undefined    │                      │
├─────────────┼───────────────┼────────────────────┤
│ ctest       │ ctest         │ ctest                │
│ (fast)      │ (2× slower)   │ (15× slower)         │
│ PASS → ship │ FAIL → block  │ FAIL → block         │
└─────────────┴───────────────┴────────────────────┘

```

---

## Self-Assessment

### Q1: Run the full test suite with ASAN+UBSAN enabled in CI and fail on any sanitizer error

**Answer:**

**CMake preset for sanitizers:**

```cmake

# CMakePresets.json (or CMakeLists.txt)
# Option 1: CMake preset
{
    "configurePresets": [{
        "name": "sanitize",
        "cacheVariables": {
            "CMAKE_CXX_FLAGS": "-fsanitize=address,undefined -fno-omit-frame-pointer -g -O1",
            "CMAKE_EXE_LINKER_FLAGS": "-fsanitize=address,undefined"
        }
    }]
}

```

```cmake

# Option 2: CMakeLists.txt with option
option(ENABLE_SANITIZERS "Enable ASAN+UBSAN" OFF)

if(ENABLE_SANITIZERS)
    add_compile_options(
        -fsanitize=address,undefined
        -fno-omit-frame-pointer
        -fno-sanitize-recover=all    # Abort on ANY error
        -g -O1
    )
    add_link_options(-fsanitize=address,undefined)
endif()

```

**GitHub Actions workflow:**

```yaml

# .github/workflows/sanitizers.yml
name: Sanitizer CI

on: [push, pull_request]

jobs:
  asan-ubsan:
    runs-on: ubuntu-latest
    env:
      # Make ASAN abort on first error (don't continue)
      ASAN_OPTIONS: "abort_on_error=1:detect_leaks=1:check_initialization_order=1"
      UBSAN_OPTIONS: "halt_on_error=1:print_stacktrace=1"
    steps:

      - uses: actions/checkout@v4

      - name: Configure with sanitizers

        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++ \
            -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined -fno-omit-frame-pointer -g -O1" \
            -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=address,undefined"

      - name: Build

        run: cmake --build build -j$(nproc)

      - name: Test

        run: |
          cd build
          ctest --output-on-failure --timeout 120
          # Exit code != 0 if ANY sanitizer error triggers

  tsan:
    runs-on: ubuntu-latest
    env:
      TSAN_OPTIONS: "halt_on_error=1:second_deadlock_stack=1"
    steps:

      - uses: actions/checkout@v4
      - name: Configure with TSAN

        run: |
          cmake -B build \
            -DCMAKE_CXX_COMPILER=clang++ \
            -DCMAKE_CXX_FLAGS="-fsanitize=thread -g -O1" \
            -DCMAKE_EXE_LINKER_FLAGS="-fsanitize=thread"

      - name: Build & Test

        run: cmake --build build -j$(nproc) && cd build && ctest --output-on-failure

```

**Key environment variables:**

| Variable | Effect |
| --- | --- |
| `ASAN_OPTIONS=abort_on_error=1` | Hard fail on memory error |
| `ASAN_OPTIONS=detect_leaks=1` | Enable leak detection |
| `UBSAN_OPTIONS=halt_on_error=1` | Abort on UB (don't just warn) |
| `UBSAN_OPTIONS=print_stacktrace=1` | Full backtrace on error |
| `-fno-sanitize-recover=all` | Compile-time: make ALL sanitizer errors fatal |

### Q2: Explain why sanitizer builds should be separate CI jobs (not the main release build)

**Answer:**

Sanitizer builds must be **separate** CI jobs for these reasons:

| Reason | Details |
| --- | --- |
| **Performance** | ASAN: 2× slower, TSAN: 5-15× slower. Release build stays fast. |
| **Binary compatibility** | Sanitizer-instrumented code can't link with non-instrumented libs safely |
| **Incompatibility** | ASAN and TSAN **cannot** be combined — they use conflicting shadow memory |
| **Optimization level** | Sanitizers work best at `-O1` or `-O0`. Release uses `-O2`/`-O3`. |
| **Memory overhead** | ASAN uses 2-3× memory, TSAN 5-10×. May exceed CI runner limits if combined. |
| **False positives at -O2** | Aggressive optimization can cause UBSAN false positives (reordering, inlining) |
| **Debug info needed** | Sanitizers need `-g` for useful stack traces — not wanted in release |
| **Different failure modes** | Release tests check correctness. Sanitizer tests check safety. Separate signals. |

**Recommended CI matrix:**

```yaml

strategy:
  matrix:
    include:

      - name: "Release"

        cmake_flags: "-O2 -DNDEBUG"

      - name: "ASAN+UBSAN"

        cmake_flags: "-fsanitize=address,undefined -fno-omit-frame-pointer -g -O1"

      - name: "TSAN"

        cmake_flags: "-fsanitize=thread -g -O1"
      # Optional:

      - name: "MSAN"

        cmake_flags: "-fsanitize=memory -fno-omit-frame-pointer -g -O1"

```

### Q3: Use sanitizer suppressions to silence known false positives in third-party libraries

**Answer:**

**ASAN suppression file:**

```cpp

# asan_suppressions.txt
# Suppress known leak in vendored library
leak:third_party/legacy_lib/init.cpp

# Suppress use-after-scope in Boost.Asio (known false positive in v1.78)
interceptor_via_fun:boost::asio::detail::*

```

**LSAN suppression file:**

```cpp

# lsan_suppressions.txt
# Known leak in third-party logging library (they cache singletons)
leak:spdlog::details::registry::instance

# OpenSSL allocates memory that's freed at exit
leak:CRYPTO_malloc
leak:SSL_library_init

```

**UBSAN suppression file:**

```cpp

# ubsan_suppressions.txt
# Third-party JSON library does intentional unsigned overflow
unsigned-integer-overflow:nlohmann/json.hpp

# Proto-buf generated code triggers alignment warning
alignment:google/protobuf/*

```

**TSAN suppression file (most common):**

```cpp

# tsan_suppressions.txt
# Known benign race in lock-free counter (intentional relaxed ordering)
race:metrics::counter::increment

# False positive in glib event loop
race:g_main_context_*

```

**Using suppressions in CI:**

```yaml

env:
  ASAN_OPTIONS: "suppressions=sanitizer/asan_suppressions.txt:abort_on_error=1"
  LSAN_OPTIONS: "suppressions=sanitizer/lsan_suppressions.txt"
  UBSAN_OPTIONS: "suppressions=sanitizer/ubsan_suppressions.txt:halt_on_error=1"
  TSAN_OPTIONS: "suppressions=sanitizer/tsan_suppressions.txt:halt_on_error=1"

```

**Best practices for suppressions:**

```cpp

// ═══════════ Source-level suppression (preferred for own code) ═══════════
// Attribute to disable a specific sanitizer for one function
__attribute__((no_sanitize("address")))
void legacy_buffer_hack(char* buf, size_t n) {
    // Known to read 1 byte past buffer — legacy protocol requirement
    char sentinel = buf[n];  // Would trigger ASAN without suppression
    // ...
}

// UBSAN: disable for specific function
__attribute__((no_sanitize("undefined")))
uint32_t hash_combine(uint32_t a, uint32_t b) {
    return a * 2654435761u + b;  // Intentional unsigned overflow
}

```

| Suppression Method | Scope | When to Use |
| --- | --- | --- |
| Suppression file | Runtime, global | Third-party code you can't modify |
| `__attribute__((no_sanitize(...)))` | Compile-time, per function | Your own code with documented reason |
| `// NOLINT` | Linter only | Not for sanitizers |
| Conditional compilation | Build-time | Different code paths for sanitizer builds |

---

## Notes

- **ASAN + UBSAN** can coexist in one build; **TSAN** must be a separate build
- Always use `-fno-sanitize-recover=all` in CI to make errors fatal
- Use `-fno-omit-frame-pointer` for complete stack traces
- MSAN requires ALL linked libraries (including libc++) to be built with MSAN — hardest to set up
- Add `symbolizer_path` to ASAN_OPTIONS if stack traces show `??` instead of function names
- Sanitizers find ~90% of memory bugs that crash in production — the most cost-effective safety net
