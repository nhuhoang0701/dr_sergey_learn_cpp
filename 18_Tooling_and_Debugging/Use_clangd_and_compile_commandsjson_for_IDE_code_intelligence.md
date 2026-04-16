# Use clangd and compile_commands.json for IDE code intelligence

**Category:** Tooling & Debugging  
**Item:** #200  
**Reference:** <https://clangd.llvm.org>  

---

## Topic Overview

clangd is a **language server** that provides IDE features (completion, diagnostics, navigation) based on the actual Clang compiler. It uses `compile_commands.json` to know the exact flags for each file.

```cpp

Editor (VS Code/Vim/Emacs)
       │ LSP protocol
    clangd
       │ reads
 compile_commands.json
  [
    { "file": "main.cpp",
      "command": "g++ -std=c++20 -I/usr/include ...",
      "directory": "/project/build" }
  ]

```

---

## Self-Assessment

### Q1: Generate `compile_commands.json` with CMake

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(MyProject LANGUAGES CXX)

# THIS IS THE KEY LINE:
set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

set(CMAKE_CXX_STANDARD 20)
add_executable(app src/main.cpp src/engine.cpp src/renderer.cpp)
target_include_directories(app PRIVATE include)

```

```bash

# Generate compile_commands.json:
cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON

# The file is in build/compile_commands.json
# Symlink it to project root for clangd to find:
ln -s build/compile_commands.json .

```

Generated `compile_commands.json`:

```json

[
  {
    "directory": "/home/user/project/build",
    "command": "/usr/bin/g++ -std=c++20 -Iinclude -o CMakeFiles/app.dir/src/main.cpp.o -c /home/user/project/src/main.cpp",
    "file": "/home/user/project/src/main.cpp"
  },
  {
    "directory": "/home/user/project/build",
    "command": "/usr/bin/g++ -std=c++20 -Iinclude -o CMakeFiles/app.dir/src/engine.cpp.o -c /home/user/project/src/engine.cpp",
    "file": "/home/user/project/src/engine.cpp"
  }
]

```

### Q2: clangd template and concept diagnostics

clangd uses the **full Clang parser**, so it can show errors that simple editors miss:

```cpp

// clangd shows this error INLINE in the editor:
#include <concepts>
#include <iostream>

template<std::integral T>  // concept constraint
T add(T a, T b) { return a + b; }

int main() {
    add(1, 2);       // OK: int satisfies std::integral
    add(1.5, 2.5);   // clangd: error! double does not satisfy std::integral
    //                // clangd message:
    //                // "no matching function for call to 'add'"
    //                // "candidate template ignored: constraints not satisfied"
    //                // "because 'std::integral<double>' evaluated to false"
}

```

clangd advantages over basic syntax highlighting:

| Feature | Basic Syntax | clangd |
| --- | --- | --- |
| Syntax errors | Some | All (Clang-accurate) |
| Template errors | None | Full diagnostics |
| Concept violations | None | Detailed constraint info |
| Include errors | None | Missing header detection |
| Go-to-definition | Text match | Semantic (correct overload) |
| Auto-complete | Keyword-based | Context-aware (types, scopes) |
| Rename | Text replace | Semantic (all references) |

### Q3: Configure `.clangd` for custom include paths

```yaml

# .clangd (place in project root)

CompileFlags:
  Add:

    - "-std=c++20"
    - "-I/opt/custom-lib/include"
    - "-I../third_party/json/include"
    - "-Wall"
    - "-Wextra"

  Remove:

    - "-W*"           # remove all warning flags from compile_commands.json

  Compiler: clang++    # use clang++ for analysis even if project uses g++

Diagnostics:
  ClangTidy:
    Add:

      - modernize-*
      - performance-*
      - bugprone-*

    Remove:

      - modernize-use-trailing-return-type  # too noisy

Index:
  Background: Build    # index in background after build

# Separate settings for test files:
---
If:
  PathMatch: tests/.*
CompileFlags:
  Add:

    - "-I../third_party/catch2/include"

```

```bash

# Verify clangd sees your configuration:
clangd --check=src/main.cpp
# Shows: flags used, errors found, time taken

```

---

## Notes

- Always symlink `compile_commands.json` to the project root.
- For non-CMake builds: use Bear (`bear -- make`) to generate compile_commands.json.
- VS Code: install "clangd" extension (disable Microsoft C/C++ IntelliSense).
- clangd indexes the project in the background for fast go-to-definition.
- Use `clangd --query-driver=/usr/bin/g++*` if using GCC system headers.
- `.clangd` config is YAML; use `---` separator for conditional sections.
