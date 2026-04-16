# Use clang-format to enforce consistent code style in a team

**Category:** Tooling & Debugging  
**Item:** #508  
**Reference:** <https://clang.llvm.org/docs/ClangFormat.html>  

---

## Topic Overview

clang-format reads a `.clang-format` file to apply consistent formatting across an entire team's codebase. Combined with CI enforcement, it eliminates code style debates.

```cpp

Workflow:
  Developer writes code
       │
  clang-format -i *.cpp  (pre-commit hook or save action)
       │
  git push
       │
  CI: clang-format --dry-run --Werror
  ┌───┴───┐
 PASS    FAIL → "Fix formatting before merge"

```

---

## Self-Assessment

### Q1: Create a `.clang-format` file and apply it across a codebase

```yaml

# .clang-format
---
BasedOnStyle: LLVM

# Team customizations:
IndentWidth: 4
TabWidth: 4
UseTab: Never
ColumnLimit: 120
AccessModifierOffset: -4
AllowShortFunctionsOnASingleLine: Empty
AlwaysBreakTemplateDeclarations: Yes
BinPackArguments: false
BinPackParameters: false
BreakBeforeBraces: Attach
NamespaceIndentation: None
PointerAlignment: Left
SortIncludes: CaseInsensitive
SpaceAfterTemplateKeyword: true
Standard: c++20
---

```

```bash

# Apply to entire codebase:
find src include tests -name '*.cpp' -o -name '*.h' | xargs clang-format -i

# Or use git ls-files to only format tracked files:
git ls-files '*.cpp' '*.h' | xargs clang-format -i

# Preview changes without modifying:
clang-format --dry-run -Werror main.cpp

```

Before:

```cpp

namespace app{class Widget {int x ;double   y;
public:
Widget( int a , double b ):x( a ),y( b ){}
void   print (
) const;
};}

```

After:

```cpp

namespace app {
class Widget {
    int x;
    double y;

public:
    Widget(int a, double b) : x(a), y(b) {}
    void print() const;
};
}  // namespace app

```

### Q2: CI step to enforce formatting

```yaml

# .github/workflows/format-check.yml
name: Formatting
on: [push, pull_request]

jobs:
  check-format:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Check clang-format

        run: |
          # Format all C++ files
          git ls-files '*.cpp' '*.h' '*.hpp' | xargs clang-format -i

          # Check if anything changed
          if ! git diff --quiet; then
            echo "::error::Code is not formatted. Run clang-format."
            git diff --stat
            git diff  # show exact changes
            exit 1
          fi
          echo "All files correctly formatted."

```

Pre-commit hook alternative:

```bash

#!/bin/bash
# .git/hooks/pre-commit
FILES=$(git diff --cached --name-only --diff-filter=ACMR | grep -E '\.(cpp|h|hpp)$')
if [ -n "$FILES" ]; then
    echo "$FILES" | xargs clang-format -i
    echo "$FILES" | xargs git add  # re-stage formatted files
fi

```

### Q3: `BasedOnStyle` vs per-option overrides

```yaml

# BasedOnStyle sets ALL options to a predefined style's defaults
# Per-option overrides then modify specific options

# Example: Start from Google style, customize a few things
---
BasedOnStyle: Google          # Sets ~120 options to Google's values
IndentWidth: 4                # Override: Google uses 2, we want 4
ColumnLimit: 100              # Override: Google uses 80
AllowShortIfStatementsOnASingleLine: Never  # Override
---

# Without BasedOnStyle, you must specify EVERY option
# clang-format has 120+ options; missing ones get LLVM defaults

# Available base styles:
#   LLVM, Google, Chromium, Mozilla, WebKit, Microsoft, GNU

```

| Base Style | Indent | Braces | Column Limit |
| --- | --- | --- | --- |
| LLVM | 2 | Attach | 80 |
| Google | 2 | Attach | 80 |
| Chromium | 2 | Attach | 80 |
| Mozilla | 2 | Linux (next line for functions) | 80 |
| WebKit | 4 | Attach | 0 (no limit) |
| Microsoft | 4 | Allman (next line) | 120 |

```bash

# Generate the full config for a base style:
clang-format --dump-config --style=google > .clang-format
# Then edit the file to customize

# Test different styles quickly:
clang-format --style=google main.cpp | diff - <(clang-format --style=mozilla main.cpp)

```

---

## Notes

- Place `.clang-format` in the project root; clang-format searches up from the file.
- Use `.clang-format-ignore` for files that shouldn't be formatted (generated code).
- `// clang-format off` / `// clang-format on` for sections that need manual formatting.
- Integrate with editors: VS Code (`formatOnSave`), Vim (`:ClangFormat`), Emacs.
- Consider adopting an established style (Google, LLVM) to minimize configuration.
