# Understand reproducible builds and eliminate non-determinism

**Category:** Build Systems & CI  
**Item:** #566  
**Reference:** <https://reproducible-builds.org/>  

---

## Topic Overview

A **reproducible build** produces bit-for-bit identical output given the same source code, build environment, and build instructions. Non-determinism sneaks in through absolute paths in debug info, timestamps embedded by `__DATE__`/`__TIME__`, and randomized data structure iteration. Eliminating these makes builds auditable, cacheable, and debuggable.

### Sources of Non-Determinism in C++ Builds

| Source | What changes | Fix |
| --- | --- | --- |
| Absolute paths in DWARF debug | `/home/alice/project/` vs `/home/bob/project/` | `-ffile-prefix-map=$(pwd)=.` |
| `__DATE__` / `__TIME__` macros | Compile timestamp | `SOURCE_DATE_EPOCH`, or ban usage |
| File ordering (glob) | Directory listing order varies | Sort file lists explicitly |
| Uninitialized padding | Struct padding bytes are random | `-ftrivial-auto-var-init=zero` |
| Parallel compilation | Object file link order | Use explicit link order in build system |
| Randomized hash seeds | `unordered_map` iteration order | Use deterministic containers in codegen |
| Compiler version | Different codegen | Pin exact compiler version |

---

## Self-Assessment

### Q1: Use -ffile-prefix-map=$(pwd)=. to strip absolute paths from debug info for reproducibility

**Answer:**

```bash

# ═══════════ The problem: absolute paths in debug info ═══════════

# When you compile with debug info:
g++ -g -o hello hello.cpp

# The DWARF debug info contains absolute paths:
readelf --debug-dump=info hello | grep DW_AT_comp_dir
# DW_AT_comp_dir: /home/alice/projects/hello
# ↑ Different on every developer's machine

# Two developers building the same code get different binaries:
# Alice: DW_AT_comp_dir: /home/alice/projects/hello
# Bob:   DW_AT_comp_dir: /home/bob/projects/hello
# sha256sum differs

# ═══════════ The fix: -ffile-prefix-map ═══════════

# Replace absolute path with relative:
g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp

readelf --debug-dump=info hello | grep DW_AT_comp_dir
# DW_AT_comp_dir:
# ↑ Same on every machine

# More specific variants:
# -fdebug-prefix-map=$(pwd)=.     # Only DWARF debug info
# -fmacro-prefix-map=$(pwd)=.     # Only __FILE__ macro expansion
# -ffile-prefix-map=$(pwd)=.      # Both (recommended)

```

```cmake

# CMakeLists.txt integration
add_compile_options(
    -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.
    -ffile-prefix-map=${CMAKE_BINARY_DIR}=build
)

# Or with CMake 3.26+ built-in support:
# set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -ffile-prefix-map=${CMAKE_SOURCE_DIR}=.")

```

```bash

# Verify the fix:
# Build 1 (in /home/alice/project/)
cd /home/alice/project && g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp
sha256sum hello
# abc123

# Build 2 (in /home/bob/project/)
cd /home/bob/project && g++ -g -ffile-prefix-map=$(pwd)=. -o hello hello.cpp
sha256sum hello
# abc123...  ← IDENTICAL

```

**Explanation:** `-ffile-prefix-map=OLD=NEW` replaces every occurrence of the `OLD` path prefix with `NEW` in the generated object files. Setting `OLD=$(pwd)` and `NEW=.` converts absolute paths to relative, making the binary identical regardless of where it was built.

### Q2: Explain how __DATE__ and __TIME__ macros break reproducibility and how to replace them

**Answer:**

```cpp

// ═══════════ The problem ═══════════
// __DATE__ and __TIME__ expand to the compilation timestamp:

#include <iostream>

void print_version() {
    std::cout << "Built on: " << __DATE__ << " " << __TIME__ << "\n";
    // Output: Built on: Jun 15 2024 14:32:01
    // ↑ Changes every time you compile!
    // Two builds from the same source → different binaries
}

```

```bash

# ═══════════ Fix 1: SOURCE_DATE_EPOCH (recommended) ═══════════
# GCC and Clang honor this environment variable:
# When set, __DATE__ and __TIME__ use this timestamp instead of "now"

export SOURCE_DATE_EPOCH=$(git log -1 --format=%ct)
# Uses the last git commit timestamp (deterministic!)

g++ -g -o hello hello.cpp
# __DATE__ is now the commit date, same on every build of this commit

# In CMake:
# set(ENV{SOURCE_DATE_EPOCH} "1718451121")  # Unix timestamp

```

```cpp

// ═══════════ Fix 2: Ban __DATE__/__TIME__ with compiler errors ═══════════

// .clang-tidy
// Checks: bugprone-suspicious-include,cert-dcl58-cpp,...

// Or use a macro that fails compilation:
#ifdef REPRODUCIBLE_BUILD
    #undef __DATE__
    #define __DATE__ "REDACTED"
    #undef __TIME__
    #define __TIME__ "REDACTED"
#endif

// ═══════════ Fix 3: Use version info from build system ═══════════

// CMakeLists.txt:
// configure_file(version.h.in version.h @ONLY)

// version.h.in:
// #define BUILD_VERSION "@PROJECT_VERSION@"
// #define BUILD_GIT_HASH "@GIT_HASH@"

#include "version.h"

void print_version_safe() {
    std::cout << "Version: " << BUILD_VERSION << "\n";
    std::cout << "Commit:  " << BUILD_GIT_HASH << "\n";
    // Deterministic! Changes only when version/commit changes
}

```

```bash

# ═══════════ Detect __DATE__/__TIME__ usage ═══════════
# GCC warning (since GCC 4.9):
g++ -Wdate-time -Werror=date-time -c hello.cpp
# error: macro "__DATE__" might prevent reproducible builds [-Werror=date-time]

# Search codebase for usage:
grep -rn '__DATE__\|__TIME__' src/ include/

```

### Q3: Verify reproducibility with diffoscope by comparing two independent builds byte-for-byte

**Answer:**

```bash

# ═══════════ Step 1: Build twice independently ═══════════

# Build 1
mkdir build1 && cd build1
cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_FLAGS="-ffile-prefix-map=$(pwd)=." \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build . --parallel $(nproc)
sha256sum myapp > ../checksums1.txt
cd ..

# Build 2 (clean directory)
mkdir build2 && cd build2
cmake .. -DCMAKE_BUILD_TYPE=Release \
    -DCMAKE_C_FLAGS="-ffile-prefix-map=$(pwd)=." \
    -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$(pwd)=."
cmake --build . --parallel $(nproc)
sha256sum myapp > ../checksums2.txt
cd ..

# ═══════════ Step 2: Compare checksums ═══════════
diff checksums1.txt checksums2.txt
# If empty (no diff) → reproducible
# If different → investigate with diffoscope

# ═══════════ Step 3: Use diffoscope for detailed comparison ═══════════
# Install:
sudo apt-get install diffoscope

# Compare binaries:
diffoscope build1/myapp build2/myapp --html report.html

# diffoscope output example:
# ── myapp
# │ ── readelf --debug-dump=info {}
# │ │ @@ -42,1 +42,1 @@
# │ │ -  DW_AT_comp_dir: /home/user/build1
# │ │ +  DW_AT_comp_dir: /home/user/build2
# │ │ ↑ Found the source of non-determinism

# ═══════════ Step 4: Fix and re-verify ═══════════
# Add -ffile-prefix-map, re-build, re-compare
# Iterate until diffoscope reports "no differences"

# ═══════════ CI automation ═══════════

```

```yaml

# .github/workflows/reproducible.yml
name: Reproducibility Check
on: [push]
jobs:
  check:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build 1

        env:
          SOURCE_DATE_EPOCH: "0"
        run: |
          cmake -B build1 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build build1
          sha256sum build1/myapp > hash1.txt

      - name: Build 2 (clean)

        env:
          SOURCE_DATE_EPOCH: "0"
        run: |
          cmake -B build2 -DCMAKE_BUILD_TYPE=Release \
            -DCMAKE_CXX_FLAGS="-ffile-prefix-map=$PWD=."
          cmake --build build2
          sha256sum build2/myapp > hash2.txt

      - name: Verify identical

        run: |
          diff hash1.txt hash2.txt
          echo "Builds are reproducible!"

```

---

## Notes

- **`strip-nondeterminism`** is a Debian tool that removes timestamps from archives (.a, .jar, .zip)
- **Link order** can be non-deterministic with CMake globs — always list source files explicitly
- **ASLR** doesn't affect binary reproducibility (it's a runtime feature), but PIE/non-PIE builds differ
- **`ar` archives** embed timestamps by default — use `ARFLAGS=Dcr` for deterministic `.a` files
- **Reproducible builds are required** for: package manager verification, supply chain security, audit compliance (ISO 27001), and efficient remote caching
