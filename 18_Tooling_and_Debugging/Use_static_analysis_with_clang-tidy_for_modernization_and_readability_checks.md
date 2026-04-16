# Use static analysis with clang-tidy for modernization and readability checks

**Category:** Tooling & Debugging  
**Item:** #422  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/extra/clang-tidy/checks/list.html>  

---

## Topic Overview

clang-tidy is a linter and static analyzer that catches bugs, enforces style, and modernizes C++ code.

| Check Category | What It Does | Examples |
| --- | --- | --- |
| `modernize-*` | Upgrade to modern C++ | `use-nullptr`, `use-override`, `use-auto` |
| `readability-*` | Improve readability | `braces-around-statements`, `identifier-naming` |
| `bugprone-*` | Find common bugs | `use-after-move`, `string-integer-assignment` |
| `performance-*` | Performance improvements | `unnecessary-copy-initialization`, `move-const-arg` |
| `cppcoreguidelines-*` | C++ Core Guidelines | `avoid-goto`, `owning-memory` |

---

## Self-Assessment

### Q1: Run clang-tidy and interpret the output

```cpp

// old_code.cpp — has modernization opportunities
#include <iostream>
#include <vector>
#include <memory>

class Base {
public:
    virtual void process() { std::cout << "Base\n"; }
    virtual ~Base() {}
};

class Derived : public Base {
public:
    void process() { std::cout << "Derived\n"; }  // missing override
    ~Derived() {}  // missing override
};

void old_style() {
    int* p = NULL;                    // should be nullptr
    Base* b = new Derived();          // should use smart pointer
    b->process();
    delete b;

    std::vector<int> v;
    for (std::vector<int>::iterator it = v.begin(); it != v.end(); ++it) {
        // should use range-for
    }

    typedef int MyInt;  // should use 'using'
}

int main() { old_style(); }

```

```bash

# Run with modernize and readability checks:
$ clang-tidy old_code.cpp \
    --checks='modernize-*,readability-*' \
    -- -std=c++20

# Output:
# old_code.cpp:12:10: warning: prefer using 'override' ... [modernize-use-override]
#     void process() {
#                    ^    override
#
# old_code.cpp:13:5: warning: prefer using 'override' ... [modernize-use-override]
#     ~Derived() {}
#     ^           override
#
# old_code.cpp:17:16: warning: use nullptr [modernize-use-nullptr]
#     int* p = NULL;
#              ^~~~  nullptr
#
# old_code.cpp:22:5: warning: use range-based for loop [modernize-loop-convert]
#     for (std::vector<int>::iterator
#     ^~~~
#     for (auto & elem : v)
#
# old_code.cpp:26:5: warning: use 'using' instead of 'typedef' [modernize-use-using]
#     typedef int MyInt;
#     ^~~~~~~~~~~~~~~~~  using MyInt = int;

```

### Q2: Auto-fix with `--fix`

```bash

# Auto-fix specific checks:
$ clang-tidy old_code.cpp \
    --checks='modernize-use-nullptr,modernize-use-override' \
    --fix \
    -- -std=c++20

# Applied fixes:
# - NULL -> nullptr
# - Added 'override' to Derived::process() and ~Derived()

```

After auto-fix:

```cpp

class Derived : public Base {
public:
    void process() override { std::cout << "Derived\n"; }  // fixed!
    ~Derived() override {}  // fixed!
};

void old_style() {
    int* p = nullptr;  // fixed!
    // ...
}

```

```bash

# Fix all modernize checks at once:
$ clang-tidy old_code.cpp --checks='modernize-*' --fix -- -std=c++20

# Dry run (show what would be fixed, don't modify):
$ clang-tidy old_code.cpp --checks='modernize-*' --fix-notes -- -std=c++20

# Fix across entire project (with compile_commands.json):
$ run-clang-tidy -p build/ -checks='modernize-*' -fix

```

### Q3: Configure `.clang-tidy` file

```yaml

# .clang-tidy — project root configuration
---
Checks: >
  -*,
  modernize-*,
  readability-*,
  bugprone-*,
  performance-*,
  -modernize-use-trailing-return-type,
  -readability-magic-numbers,
  -readability-identifier-length

WarningsAsErrors: >
  bugprone-*,
  modernize-use-override

HeaderFilterRegex: 'src/.*\.h$'

CheckOptions:

  - key: readability-identifier-naming.ClassCase

    value: CamelCase

  - key: readability-identifier-naming.FunctionCase

    value: camelBack

  - key: readability-identifier-naming.VariableCase

    value: lower_case

  - key: readability-identifier-naming.ConstantCase

    value: UPPER_CASE

  - key: modernize-use-auto.MinTypeNameLength

    value: 10

  - key: readability-function-cognitive-complexity.Threshold

    value: 25

```

```bash

# Run using .clang-tidy config (auto-detected from file):
$ clang-tidy src/*.cpp -- -std=c++20

# CI integration:
$ clang-tidy --warnings-as-errors='*' src/*.cpp -- -std=c++20
# Returns non-zero exit code on any warning -> CI fails

# CMake integration:
set(CMAKE_CXX_CLANG_TIDY
    clang-tidy
    --config-file=${CMAKE_SOURCE_DIR}/.clang-tidy
)

```

---

## Notes

- Start with `modernize-*` and `bugprone-*`; add more gradually.
- `readability-identifier-naming` enforces naming conventions across the project.
- Use `// NOLINT(check-name)` to suppress specific warnings on a line.
- `run-clang-tidy` parallelizes analysis across files.
- clang-tidy requires `compile_commands.json` for accurate analysis.
