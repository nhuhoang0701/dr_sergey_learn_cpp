# Use compile_commands.json and Bear for IDE integration

**Category:** Build Systems & CI  
**Item:** #666  
**Reference:** <https://clangd.llvm.org/installation>  

---

## Topic Overview

`compile_commands.json` is a **JSON compilation database** — it tells tools (clangd, clang-tidy, include-what-you-use) the exact compiler invocation for every source file. This is what powers accurate code navigation, auto-complete, and diagnostics in modern C++ IDEs.

### Why It Matters

```cpp

Without compile_commands.json:
  IDE doesn't know -I/path/to/headers or -DFEATURE_X=1
  → Red squiggles on valid #include
  → Missing auto-complete for project-specific macros
  → clang-tidy can't analyze your code

With compile_commands.json:
  IDE knows every flag for every file
  → Accurate navigation, completion, diagnostics

```

### Generation Methods

| Method | Works With | Command |
| --- | --- | --- |
| CMake | CMake projects | `-DCMAKE_EXPORT_COMPILE_COMMANDS=ON` |
| Bear | Make, autotools, any build | `bear -- make` |
| compdb | Ninja | `ninja -t compdb` |
| intercept-build | Any build system | `intercept-build make` |

---

## Self-Assessment

### Q1: Generate compile_commands.json with CMake and verify clangd picks up include paths

**Answer:**

```bash

# ═══════════ Generate with CMake ═══════════
cmake -B build -G Ninja -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
# Or add to CMakeLists.txt:
#   set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

# Result: build/compile_commands.json is created
cat build/compile_commands.json

```

```json

[
    {
        "directory": "/project/build",
        "command": "/usr/bin/g++ -std=c++20 -Wall -Wextra -I/project/include -I/project/third_party/fmt/include -DVERSION=\\\"2.1.0\\\" -o CMakeFiles/app.dir/src/main.cpp.o -c /project/src/main.cpp",
        "file": "/project/src/main.cpp"
    },
    {
        "directory": "/project/build",
        "command": "/usr/bin/g++ -std=c++20 -Wall -Wextra -I/project/include -o CMakeFiles/mathlib.dir/src/math.cpp.o -c /project/src/math.cpp",
        "file": "/project/src/math.cpp"
    }
]

```

```bash

# ═══════════ Symlink to project root (clangd looks there) ═══════════
ln -sf build/compile_commands.json compile_commands.json
# Or: add to .gitignore and let each dev create the symlink

# ═══════════ Verify clangd picks it up ═══════════
clangd --check=src/main.cpp
# Output:
# I[...] Loaded compilation database from /project/compile_commands.json
# I[...] ASTWorker building file /project/src/main.cpp
# I[...] Include search path: /project/include
# I[...] Include search path: /project/third_party/fmt/include
# No errors

# ═══════════ VS Code setup ═══════════
# Install clangd extension, add to settings.json:
# {
#     "clangd.arguments": [
#         "--compile-commands-dir=${workspaceFolder}/build",
#         "--header-insertion=iwyu",
#         "--clang-tidy"
#     ]
# }

```

### Q2: Use Bear to generate compile_commands.json for Make-based projects that lack CMake support

**Answer:**

```bash

# ═══════════ Install Bear ═══════════
# Ubuntu/Debian:
sudo apt install bear
# macOS:
brew install bear
# Arch:
sudo pacman -S bear

# ═══════════ Generate for a Make project ═══════════
# Bear intercepts compiler calls during the build:
bear -- make -j$(nproc)
# Result: compile_commands.json in current directory

# For a clean rebuild (ensures all files are captured):
make clean
bear -- make -j$(nproc)

# ═══════════ Bear with autotools ═══════════
bear -- ./configure && bear --append -- make

# ═══════════ Bear with custom build script ═══════════
bear -- ./build.sh

# ═══════════ Verify the output ═══════════
cat compile_commands.json
# [
#   {
#     "directory": "/project",
#     "arguments": ["/usr/bin/g++", "-std=c++20", "-Wall",
#                   "-I./include", "-c", "src/main.cpp",
#                   "-o", "obj/main.o"],
#     "file": "src/main.cpp"
#   },
#   
# ]

# ═══════════ How Bear works internally ═══════════
# 1. Sets LD_PRELOAD to intercept exec() calls
# 2. Watches for compiler invocations (g++, clang++, cc, c++)
# 3. Records every compiler call with full arguments
# 4. Writes compile_commands.json after build finishes

# ═══════════ Alternative: Ninja's built-in compdb ═══════════
ninja -t compdb cxx > compile_commands.json
# Works without Bear, but only for Ninja-based builds

```

### Q3: Configure a .clangd file to override compiler flags for a specific directory

**Answer:**

```yaml

# .clangd — placed in project root
# This file overrides/supplements flags from compile_commands.json

# Global settings
CompileFlags:
  Add:

    - -Wall
    - -Wextra
    - -Wpedantic
    - -std=c++20

  Remove:

    - -W*              # Remove all existing warning flags (re-add above)
    - -march=*         # Remove arch-specific flags for portability

Diagnostics:
  ClangTidy:
    Add:

      - modernize-*
      - bugprone-*
      - performance-*

    Remove:

      - modernize-use-trailing-return-type

InlayHints:
  Enabled: Yes
  ParameterNames: Yes
  DeducedTypes: Yes

---
# Per-directory overrides using If blocks
If:
  PathMatch: tests/.*
CompileFlags:
  Add:

    - -DTESTING=1
    - -fsanitize=address,undefined

---
If:
  PathMatch: third_party/.*
CompileFlags:
  Add:

    - -w              # Suppress all warnings in third-party code

Diagnostics:
  Suppress: '*'       # No diagnostics for third-party headers

---
If:
  PathMatch: src/legacy/.*
CompileFlags:
  Add:

    - -std=c++17      # Legacy code still uses C++17

  Remove:

    - -std=c++20

```

```bash

# Verify .clangd is effective:
clangd --check=tests/test_main.cpp 2>&1 | grep "flags"
# Should show: -DTESTING=1 -fsanitize=address,undefined

clangd --check=third_party/json.hpp 2>&1 | grep "warnings"
# Should show: warnings suppressed

```

---

## Notes

- **Symlink trick:** `ln -sf build/compile_commands.json .` — many tools look in the project root
- **CMakePresets.json:** add `"CMAKE_EXPORT_COMPILE_COMMANDS": "ON"` to your base preset's cacheVariables
- **Header files** are not in `compile_commands.json` — clangd infers their flags from the `.cpp` files that include them
- **Merge databases:** `jq -s 'add' db1.json db2.json > compile_commands.json` when combining multiple build directories
