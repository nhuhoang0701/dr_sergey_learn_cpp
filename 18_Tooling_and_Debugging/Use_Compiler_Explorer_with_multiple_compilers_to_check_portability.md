# Use Compiler Explorer with multiple compilers to check portability

**Category:** Tooling & Debugging  
**Item:** #803  
**Standard:** C++20  
**Reference:** <https://godbolt.org>  

---

## Topic Overview

Compiler Explorer supports 100+ compilers. Testing code against GCC, Clang, and MSVC simultaneously catches portability bugs before they reach CI.

```cpp

Godbolt layout for portability testing:
┌────────────┬────────────┬────────────┬────────────┐
│  Source    │ GCC 14     │ Clang 18   │ MSVC 19.38 │
│  (shared)  │ -std=c++20 │ -std=c++20 │ /std:c++20 │
│            │ OK ✓       │ OK ✓       │ error ✗    │
└────────────┴────────────┴────────────┴────────────┘

```

---

## Self-Assessment

### Q1: Find compiler-specific warnings by comparing GCC and Clang

```cpp

#include <iostream>
#include <cstring>

// Code that triggers different warnings on different compilers:

void example() {
    // 1. Narrowing in list initialization:
    int x = 1000;
    char c = x;  // GCC: -Wnarrowing (sometimes), Clang: -Wconstant-conversion

    // 2. Implicit fallthrough:
    switch (x) {
    case 1:
        std::cout << "one";
        // GCC: -Wimplicit-fallthrough (C++17 with [[fallthrough]])
        // Clang: -Wimplicit-fallthrough
    case 2:
        std::cout << "two";
        break;
    }

    // 3. Comparison of different signedness:
    unsigned u = 5;
    int i = -1;
    if (i < u) {}  // Both warn with -Wsign-compare
                   // But Clang also warns with -Wsign-conversion

    // 4. GCC-only: -Wstrict-aliasing
    float f = 3.14f;
    int* pi = reinterpret_cast<int*>(&f);  // GCC warns; UB in both

    // 5. Clang-only: -Wself-assign
    int val = 42;
    val = val;  // Clang: -Wself-assign, GCC: silent

    (void)c; (void)pi; (void)val;
}

int main() {
    example();
    std::cout << "Compiled successfully\n";
}

```

### Q2: Use the conformance view to check feature support

Godbolt's "Add new…" → "Conformance" view tests code against many compilers at once:

```cpp

// Feature: C++23 std::expected
#include <expected>

std::expected<int, std::string> parse(const char* s) {
    if (s[0] >= '0' && s[0] <= '9')
        return s[0] - '0';
    return std::unexpected("not a digit");
}

int main() {
    auto r = parse("5");
    return r.value_or(-1);
}

```

Conformance view result:

```text

GCC 12:        ✗ (no <expected> yet)
GCC 13:        ✓
Clang 16:      ✗ (partial)
Clang 17:      ✓
MSVC 19.36:    ✓
MSVC 19.34:    ✗

```

Other features to test:

```cpp

// C++20 features with variable support:
auto f(auto x) { return x; }         // abbreviated template (GCC10+, Clang12+, MSVC19.28+)
auto [a, b] = std::pair{1, 2};        // structured bindings (C++17, widely supported)
consteval int sq(int n) { return n*n; }  // consteval (GCC10+, Clang11+, MSVC19.29+)

```

### Q3: Share a permlink for a compiler bug report

```cpp

// Potential GCC bug: internal compiler error on valid code
// Minimize the reproduction first!

template<typename T>
concept Addable = requires(T a, T b) { a + b; };

template<Addable T>
auto add(T a, T b) { return a + b; }

// If this ICEs on GCC but compiles on Clang:
int main() {
    return add(1, 2);
}

```

**Steps to report:**

1. Paste code in Godbolt with the failing compiler version.
2. Click **Share** → **Short Link** (e.g., `https://godbolt.org/z/abc123`).
3. File bug at gcc.gnu.org/bugzilla or github.com/llvm/llvm-project/issues.
4. Include:
   - Permlink to Godbolt
   - Compiler version (e.g., "GCC 14.1 x86-64")
   - Expected behavior ("should compile") vs actual ("ICE")
   - Note which compilers succeed ("Clang 18 and MSVC 19.38 accept this")

```cpp

Bug report template:
  Title: [C++20] ICE on valid concept-constrained function template
  Compiler: GCC 14.1 (x86-64)
  Flags: -std=c++20 -O2
  Godbolt: https://godbolt.org/z/abc123
  Expected: Compiles successfully
  Actual: internal compiler error: in ...
  Note: Clang 18 and MSVC 19.38 accept this code.

```

---

## Notes

- Add multiple compiler panes for side-by-side comparison.
- Use "Conformance" view for quick feature-support checks across all compilers.
- Permlinks are permanent and stable — safe for bug reports and documentation.
- Test with both GCC and Clang since they have different C++20/23 support timelines.
- MSVC uses `/std:c++20` and `/std:c++latest` (not `-std=c++20`).
