# Use Bazel or Buck2 as an alternative to CMake for large-scale builds

**Category:** Build Systems & CI  
**Item:** #569  
**Reference:** <https://bazel.build/>  

---

## Topic Overview

**Bazel** (Google) and **Buck2** (Meta) are build systems designed for massive monorepos with thousands of targets. Unlike CMake+Ninja (which relies on convention and implicit dependencies), Bazel and Buck2 enforce **explicit dependency declarations** and **hermetic sandboxed builds**, guaranteeing correctness and enabling powerful caching and remote execution.

### Build System Comparison

| Feature | CMake + Ninja | Bazel | Buck2 |
| --- | --- | --- | --- |
| Dependency model | Implicit (headers) | Explicit (BUILD files) | Explicit (BUCK files) |
| Reproducibility | Manual effort | Built-in (sandbox) | Built-in (sandbox) |
| Remote caching | sccache/ccache | Native | Native |
| Remote execution | None | Native | Native (RE) |
| Incremental builds | File timestamps | Content hash | Content hash |
| Scaling | ~100K files | Millions of files | Millions of files |
| Learning curve | Low | High | High |
| Ecosystem | Dominant in C++ | Growing | Meta-focused |

---

## Self-Assessment

### Q1: Write a BUILD file that defines a cc_library and cc_binary with explicit dependency edges

**Answer:**

```python

# BUILD file (Starlark syntax — Python-like)
# Every source file and dependency must be explicitly listed

# ═══════════ Library target ═══════════
cc_library(
    name = "math_utils",
    srcs = ["math_utils.cpp"],
    hdrs = ["math_utils.h"],
    deps = [
        "@abseil-cpp//absl/strings",       # External dependency
        "//common:logging",                  # Internal dependency from //common/
    ],
    visibility = ["//visibility:public"],   # Other packages can depend on this
    copts = ["-std=c++20"],
)

# ═══════════ Binary target ═══════════
cc_binary(
    name = "calculator",
    srcs = ["main.cpp"],
    deps = [
        ":math_utils",                       # Same-package dependency
        "//ui:terminal_ui",                  # Cross-package dependency
        "@fmt//:fmt",                        # External: fmtlib
    ],
    copts = ["-std=c++20", "-Wall", "-Wextra"],
    linkopts = ["-lpthread"],
)

# ═══════════ Test target ═══════════
cc_test(
    name = "math_utils_test",
    srcs = ["math_utils_test.cpp"],
    deps = [
        ":math_utils",
        "@googletest//:gtest_main",
    ],
    copts = ["-std=c++20"],
    size = "small",  # Test size hint: small (<1min), medium, large
)

```

```python

# WORKSPACE file (root of the project) — declares external dependencies
workspace(name = "my_project")

load("@bazel_tools//tools/build_defs/repo:http.bzl", "http_archive")

# Google Test
http_archive(
    name = "googletest",
    urls = ["https://github.com/google/googletest/archive/refs/tags/v1.14.0.tar.gz"],
    strip_prefix = "googletest-1.14.0",
    sha256 = "...",
)

# Abseil
http_archive(
    name = "abseil-cpp",
    urls = ["https://github.com/abseil/abseil-cpp/archive/refs/tags/20240116.0.tar.gz"],
    strip_prefix = "abseil-cpp-20240116.0",
    sha256 = "...",
)

```

```bash

# Build and run:
bazel build //path/to:calculator
bazel run //path/to:calculator -- --input data.txt

# Run tests:
bazel test //path/to:math_utils_test
bazel test //...  # Run ALL tests in the repo

# Query the dependency graph:
bazel query 'deps(//path/to:calculator)' --output graph | dot -Tpng > deps.png

```

**Key differences from CMake:**

- Every `.cpp` file must be listed in `srcs` — no globs (intentional!)
- Every header dependency must be in `hdrs` or `deps` — no implicit include paths
- Dependencies are directed edges: if A depends on B, you must write `deps = [":B"]`

### Q2: Explain Bazel's hermetic build sandbox and how it guarantees dependency correctness

**Answer:**

```cpp

═══════════ Bazel's Sandbox Model ═══════════

Normal build (CMake/Make):
  compiler sees: entire filesystem

  - Can #include any header anywhere
  - Can link against any library
  - Missing dep? Might still work by accident

Bazel sandboxed build:
  compiler sees: ONLY declared inputs
  ┌────────────────────────────────────┐
  │ Sandbox (temporary directory)       │
  │ ├── math_utils.cpp     (from srcs) │
  │ ├── math_utils.h       (from hdrs) │
  │ ├── absl/strings.h     (from deps) │
  │ └── (nothing else!)                │
  └────────────────────────────────────┘
  
  If math_utils.cpp does #include "missing.h":
    → Compile ERROR! "missing.h" not found
    → Even if missing.h exists in the repo
    → Because it wasn't declared in deps/hdrs

```

```bash

# How sandboxing works on Linux:
# 1. Bazel creates a temporary directory
# 2. Symlinks only the declared inputs into it
# 3. Runs the compiler inside a Linux namespace (chroot-like)
# 4. The compiler can ONLY see what was declared

# Example of a sandboxing failure (caught correctly):
# If you forget to add a dependency:

# BUILD:
# cc_library(
#     name = "parser",
#     srcs = ["parser.cpp"],
#     hdrs = ["parser.h"],
#     # MISSING: deps = ["//utils:string_helpers"]
# )

# parser.cpp:
# #include "utils/string_helpers.h"  ← ERROR: not in sandbox

# Bazel output:
# ERROR: parser.cpp:1: 'utils/string_helpers.h' file not found
# NOTE: add deps = ["//utils:string_helpers"] to your BUILD target

# This GUARANTEES that the dependency graph is complete and correct
# → No "works on my machine" because I had a stale header in my include path
# → No silent dependency on accidentally-available system header

# ═══════════ Why this matters for correctness ═══════════
# Content-addressed caching relies on knowing ALL inputs
# If an input is missing from the declaration:
#   - Cache entry might be stale (input changed but cache doesn't know)
#   - Remote build might fail (different machine, different environment)
# Sandboxing prevents this class of bugs entirely

```

### Q3: Compare Bazel remote execution with CMake+Ninja for a 1000-TU project build time

**Answer:**

```cpp

═══════════ Build Time Comparison: 1000 Translation Units ═══════════

Scenario: 1000 .cpp files, 200 libraries, 50 test targets

┌─────────────────────┬──────────────────┬─────────────────────────┐
│ Build Type           │ CMake + Ninja    │ Bazel (remote exec)     │
├─────────────────────┼──────────────────┼─────────────────────────┤
│ Clean build (local)  │ 30 min (8 cores) │ 35 min (8 cores)       │
│ Clean build (remote) │ N/A              │ 5 min (200 remote CPUs) │
│ 1 file changed       │ 30 sec           │ 15 sec                  │
│ 10 files changed     │ 2 min            │ 20 sec (parallel exec)  │
│ No change rebuild    │ 5 sec (Ninja)    │ 2 sec (cached)          │
│ New developer clone  │ 30 min           │ 5 min (cached artifacts)│
│ CI (warm cache)      │ 5-10 min         │ 1-3 min                 │
└─────────────────────┴──────────────────┴─────────────────────────┘

```

```bash

# ═══════════ Bazel Remote Execution ═══════════
# .bazelrc configuration:
# build --remote_executor=grpc://buildserver.example.com:8980
# build --remote_cache=grpc://buildserver.example.com:8980
# build --jobs=200  # 200 parallel actions on remote workers

# What happens on "bazel build //...":
# 1. Bazel computes the action graph (1000 compile + 200 link actions)
# 2. For each action, checks remote cache
#    - Cache hit: download result (milliseconds)
#    - Cache miss: send action + inputs to remote worker
# 3. Remote workers compile in parallel (up to 200 simultaneously)
# 4. Results cached for future builds
# 5. Only the final binary is downloaded to the developer machine

# CMake + Ninja equivalent:
# cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
# cmake --build build -j$(nproc)
# → Limited to local CPU count
# → No remote execution (sccache helps with compilation caching only)
# → No action-level parallelism across machines

# ═══════════ When is Bazel worth the migration cost? ═══════════

```

| Factor | CMake better | Bazel better |
| --- | --- | --- |
| Project size | < 100 TUs | > 500 TUs |
| Team size | 1-5 developers | 10+ developers |
| Build frequency | Low (manual) | High (CI/CD) |
| Monorepo | Single project | Multi-project monorepo |
| Setup cost | Low | High (need BUILD files everywhere) |
| Ecosystem | Most C++ libs use CMake | Google/Meta ecosystem |
| Remote execution | Not available | Massive speedup |

**Bottom line:** For projects under ~100 TUs with a small team, CMake+Ninja is simpler and fast enough. For monorepos with 500+ TUs, frequent CI, and multiple teams, Bazel's remote execution and guaranteed correctness pay for the migration cost.

---

## Notes

- **Buck2** (Meta) is similar to Bazel but uses a different configuration language (Starlark variant) and is optimized for Meta's monorepo
- **Bazel modules (bzlmod)** is the modern dependency management system, replacing WORKSPACE files
- **CMake-to-Bazel migration** tools exist: `cmake_external` rule in Bazel can wrap CMake projects
- **Please** (pleasebuild.com) is another alternative that aims to be simpler than Bazel
- **Ninja** is a build executor, not a build system — it's analogous to Bazel's action execution layer, not to Bazel's full dependency management
