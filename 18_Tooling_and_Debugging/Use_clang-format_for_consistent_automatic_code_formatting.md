# Use clang-format for consistent automatic code formatting

**Category:** Tooling & Debugging  
**Item:** #411  
**Reference:** <https://clang.llvm.org/docs/ClangFormat.html>  

---

## Topic Overview

clang-format automatically formats C++ code according to a style configuration. It eliminates code style debates and ensures consistency across a team.

```bash

# Format a file in-place:
clang-format -i main.cpp

# Check without modifying (CI mode):
clang-format --dry-run --Werror main.cpp

# Format all C++ files recursively:
find . -name '*.cpp' -o -name '*.h' | xargs clang-format -i

```

---

## Self-Assessment

### Q1: Create a `.clang-format` configuration file

```yaml

# .clang-format (place in project root)
---
Language: Cpp
BasedOnStyle: Google  # Start from a well-known base

# Customizations:
IndentWidth: 4
ColumnLimit: 100
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: Inline
AlwaysBreakTemplateDeclarations: Yes
BreakBeforeBraces: Attach
PointerAlignment: Left
SortIncludes: CaseInsensitive
IncludeBlocks: Regroup
IncludeCategories:

  - Regex:    '<[^/]+>'      # system headers

    Priority: 3

  - Regex:    '<.+>'         # third-party headers

    Priority: 2

  - Regex:    '".*"'         # project headers

    Priority: 1
---

```

Before formatting:

```cpp

#include<iostream>
#include "myheader.h"
#include<vector>
class Widget{
    int x;double y;
public:
Widget(int a,double b):x(a),y(b){}
void print(){std::cout<<x<<" "<<y<<"\n";}
};
int main(){Widget w(1,2.0);w.print();}

```

After `clang-format -i main.cpp`:

```cpp

#include "myheader.h"

#include <iostream>
#include <vector>

class Widget {
    int x;
    double y;

public:
    Widget(int a, double b) : x(a), y(b) {}
    void print() { std::cout << x << " " << y << "\n"; }
};

int main() {
    Widget w(1, 2.0);
    w.print();
}

```

### Q2: CI step to enforce formatting

```yaml

# .github/workflows/format.yml
name: Format Check

on: [push, pull_request]

jobs:
  clang-format:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install clang-format

        run: sudo apt-get install -y clang-format-17

      - name: Check formatting

        run: |
          find . -name '*.cpp' -o -name '*.h' | \
            xargs clang-format-17 --dry-run --Werror
        # --dry-run: don't modify files
        # --Werror: exit with error if changes needed
        # If formatting is wrong, CI fails with a diff-like output

```

Alternative: use `git diff` to show exact changes:

```bash

# Format and check for diff
find . -name '*.cpp' -o -name '*.h' | xargs clang-format-17 -i
git diff --exit-code
# If there's a diff, files weren't formatted -> fail CI

```

### Q3: Handling macro-heavy code and `// clang-format off`

```cpp

#include <iostream>

// clang-format mangles certain macro patterns:

// Example: DSL-like macros with custom formatting
// clang-format off
#define REGISTER_TEST(name, func) \
    static TestEntry entry_##name = { \
        .test_name = #name,           \
        .test_func = func,            \
        .enabled   = true             \
    }
// clang-format on

// Lookup tables with intentional alignment:
// clang-format off
const int OPCODES[] = {
    0x00, 0x01, 0x02, 0x03,
    0x10, 0x11, 0x12, 0x13,
    0x20, 0x21, 0x22, 0x23,
};
// clang-format on

// Matrix literals:
// clang-format off
const float IDENTITY_4X4[] = {
    1.0f, 0.0f, 0.0f, 0.0f,
    0.0f, 1.0f, 0.0f, 0.0f,
    0.0f, 0.0f, 1.0f, 0.0f,
    0.0f, 0.0f, 0.0f, 1.0f,
};
// clang-format on

int main() {
    std::cout << "Opcode[0]: " << OPCODES[0] << '\n';
    std::cout << "Identity[0]: " << IDENTITY_4X4[0] << '\n';
}
// Expected output:
// Opcode[0]: 0
// Identity[0]: 1

```

**When to use `// clang-format off`:**

- Lookup tables with intentional column alignment
- Macro definitions with custom backslash alignment
- Matrix/vector literals
- Assembly blocks
- Generated code (embed raw data)

---

## Notes

- Generate a config interactively: `clang-format --dump-config --style=google > .clang-format`.
- Popular base styles: `LLVM`, `Google`, `Chromium`, `Mozilla`, `WebKit`, `Microsoft`.
- VS Code: install "C/C++" extension; set `C_Cpp.clang_format_style` to `file`.
- Use `.clang-format-ignore` to exclude files (e.g., generated code, third-party).
- `clang-format` can also format Java, JavaScript, JSON, Proto files.
