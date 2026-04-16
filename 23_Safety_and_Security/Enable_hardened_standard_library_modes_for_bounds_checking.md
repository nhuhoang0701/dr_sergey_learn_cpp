# Enable hardened standard library modes for bounds checking

**Category:** Safety & Security  
**Item:** #558  
**Reference:** <https://libcxx.llvm.org/Hardening.html>  

---

## Topic Overview

This topic covers libstdc++ debug mode and libc++ hardening for bounds checking — complementing #650 (debug builds focus) and #735 (iterator validity checks).

```cpp

Standard library hardening levels:

  Release (no checks):   v[10] on size-5 vector -> UB (silent corruption)
  Hardened (libc++):     v[10] -> assertion failure (abort)
  Debug (libstdc++):     v[10] -> detailed error message + abort

  Compile flags:
    libstdc++: -D_GLIBCXX_DEBUG [-D_GLIBCXX_DEBUG_PEDANTIC]
    libc++:    -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST  (Clang 18+)
               -D_LIBCPP_ENABLE_HARDENED_MODE=1  (Clang 17)

```

| Mode | Library | Checks | Overhead | Use |
| --- | --- | --- | --- | --- |
| None | Any | No bounds checks | 0% | Release (risky) |
| `_GLIBCXX_DEBUG` | libstdc++ | Full: bounds, iterators, algorithms | 50-300% | Debug/test |
| `_GLIBCXX_ASSERTIONS` | libstdc++ | Basic: bounds, null | 5-10% | Production OK |
| `_LIBCPP_HARDENING_MODE_FAST` | libc++ | Preconditions, bounds | 1-5% | Production |
| `_LIBCPP_HARDENING_MODE_EXTENSIVE` | libc++ | + semantic checks | 10-30% | Staging |
| `_LIBCPP_HARDENING_MODE_DEBUG` | libc++ | + iterator tracking | 50-200% | Debug |

---

## Self-Assessment

### Q1: _GLIBCXX_DEBUG bounds checking

```cpp

// Compile: g++ -std=c++20 -D_GLIBCXX_DEBUG bounds.cpp
#include <iostream>
#include <vector>
#include <string>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // Normal access (ok)
    std::cout << "v[2] = " << v[2] << '\n';

    // .at() always checks (even without debug mode)
    try {
        v.at(10);  // throws std::out_of_range
    } catch (const std::out_of_range& e) {
        std::cout << ".at(10): " << e.what() << '\n';
    }

    // operator[] with _GLIBCXX_DEBUG:
    // Without _GLIBCXX_DEBUG:  v[10] -> undefined behavior (silent)
    // With _GLIBCXX_DEBUG:     v[10] -> debug assertion failure + abort
    //   Error: attempt to subscript container with out-of-bounds index 10,
    //   but container only holds 5 elements.

    // std::cout << v[10];  // Uncomment to test: crashes with _GLIBCXX_DEBUG

    // Iterator invalidation detection
    std::vector<int> w = {1, 2, 3};
    auto it = w.begin();
    w.push_back(4);  // May invalidate iterators!

    // Without _GLIBCXX_DEBUG: *it -> UB (dangling iterator)
    // With _GLIBCXX_DEBUG: *it -> assertion: "attempt to dereference
    //   a past-the-end iterator" or invalidated iterator error

    // std::cout << *it;  // Uncomment: crashes with _GLIBCXX_DEBUG

    std::cout << "\n_GLIBCXX_DEBUG checks:\n";
    std::cout << "  - vector::operator[] bounds\n";
    std::cout << "  - Iterator validity after modification\n";
    std::cout << "  - Algorithm preconditions (sorted ranges)\n";
    std::cout << "  - String bounds and null pointer\n";
}

```

### Q2: libc++ hardened mode

```cpp

// Compile with Clang/libc++:
// clang++ -std=c++20 -stdlib=libc++ \
//   -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST hardened.cpp

#include <iostream>
#include <vector>
#include <span>
#include <optional>

int main() {
    // libc++ hardened mode checks preconditions at minimal cost

    std::vector<int> v = {10, 20, 30};

    // These are checked in hardened mode:
    // v[5];       // FAST: assertion failure (bounds check)
    // v.front();  // on empty vector: assertion failure
    // v.back();   // on empty vector: assertion failure

    // span bounds checking
    std::span<int> s(v);
    // s[10];  // assertion failure in hardened mode

    // optional value access
    std::optional<int> opt;
    // *opt;   // assertion failure (no value)

    std::cout << "libc++ hardening modes (Clang 18+):\n";
    std::cout << "  FAST:      bounds, null, empty -> ~1-5% overhead\n";
    std::cout << "  EXTENSIVE: + semantic (e.g., sorted input) -> ~10-30%\n";
    std::cout << "  DEBUG:     + iterator tracking -> ~50-200%\n";
    std::cout << "  NONE:      no checks (default release)\n";

    std::cout << "\nRecommendation:\n";
    std::cout << "  Production: FAST (cheap, catches common bugs)\n";
    std::cout << "  CI/testing: EXTENSIVE or DEBUG\n";
    std::cout << "  Never: NONE in security-critical code\n";
}

```

### Q3: Performance cost analysis

| Hardening level | Typical overhead | What's checked | Suitable for |
| --- | --- | --- | --- |
| None | 0% | Nothing | Perf-critical, trusted input |
| `_GLIBCXX_ASSERTIONS` | 5-10% | operator[], front/back, null | Production |
| `_LIBCPP_HARDENING_MODE_FAST` | 1-5% | Preconditions, bounds | Production |
| `_GLIBCXX_DEBUG` | 50-300% | Everything + iterators | Debug/test only |
| `_LIBCPP_HARDENING_MODE_DEBUG` | 50-200% | Everything + iterators | Debug/test only |

**Why even 1-5% is worth it:**

- Bounds violations are the #1 source of CVEs in C/C++ code.
- CWE-125 (Out-of-bounds Read) and CWE-787 (Out-of-bounds Write) are #1 and #2 on CWE Top 25.
- A 1-5% slowdown to prevent remote code execution is an excellent tradeoff.

**When to use what:**

```cpp

Debug build:      -D_GLIBCXX_DEBUG -O0 -g  (full checks)
CI/test build:    -D_GLIBCXX_ASSERTIONS -O2 -g  (moderate checks)
Release build:    -D_GLIBCXX_ASSERTIONS -O2  (or FAST for libc++)
Ultra-perf build: no hardening -O3  (only for measured bottlenecks)

```

---

## Notes

- Complementary to #650 (debug build focus) and #735 (iterator validity).
- `_GLIBCXX_DEBUG` changes ABI! Don't mix debug/release object files.
- `_GLIBCXX_ASSERTIONS` does NOT change ABI — safe for partial adoption.
- MSVC: `/sdl` enables bounds checking for `std::vector::operator[]` in debug mode.
- Consider AddressSanitizer (`-fsanitize=address`) alongside hardened STL for testing.
