# Enable hardened STL mode for bounds-checked containers in debug builds

**Category:** Safety & Security  
**Item:** #650  
**Reference:** <https://libcxx.llvm.org/Hardening.html>  

---

## Topic Overview

This topic focuses on debug-build hardening with out-of-bounds examples — complementing #558 (production hardening + performance analysis) and #735 (iterator validity + mode comparison).

```cpp

Out-of-bounds access detection:

  Without hardening:           With hardening:
  v = {1, 2, 3}               v = {1, 2, 3}
  v[5] -> ???                  v[5] -> ABORT!
  
  Release: reads garbage       Debug mode:
  or crashes randomly          "vector::operator[]: index 5
  or silently corrupts          out of range for container
  memory -> RCE exploit         of size 3"

```

---

## Self-Assessment

### Q1: _GLIBCXX_DEBUG for bounds checking

```cpp

// Compile: g++ -std=c++20 -D_GLIBCXX_DEBUG -g -O0 debug_vector.cpp
#include <iostream>
#include <vector>
#include <string>
#include <algorithm>

int main() {
    // === Vector bounds checking ===
    std::vector<int> v = {10, 20, 30};

    std::cout << "Safe: v[0] = " << v[0] << '\n';
    std::cout << "Safe: v[2] = " << v[2] << '\n';

    // This crashes with _GLIBCXX_DEBUG:
    // std::cout << v[5];  // Error: index 5, size 3

    // === String bounds checking ===
    std::string s = "hello";
    std::cout << "Safe: s[0] = " << s[0] << '\n';
    // std::cout << s[100];  // Error: index 100, size 5

    // === Algorithm precondition checking ===
    std::vector<int> unsorted = {3, 1, 2};
    // std::binary_search(unsorted.begin(), unsorted.end(), 2);
    // Error: elements not sorted! (requires sorted range)

    // === Empty container access ===
    std::vector<int> empty;
    // empty.front();  // Error: front called on empty container
    // empty.back();   // Error: back called on empty container

    std::cout << "\nAll _GLIBCXX_DEBUG checks:\n";
    std::cout << "  vector/string/deque operator[]\n";
    std::cout << "  front()/back() on empty\n";
    std::cout << "  Iterator past-the-end dereference\n";
    std::cout << "  Invalidated iterator use\n";
    std::cout << "  Algorithm preconditions (sorted, etc.)\n";
}

```

### Q2: _LIBCPP_ENABLE_HARDENED_MODE for libc++

```cpp

// Compile: clang++ -std=c++20 -stdlib=libc++ \
//   -D_LIBCPP_HARDENING_MODE=_LIBCPP_HARDENING_MODE_FAST hardened.cpp

#include <iostream>
#include <vector>
#include <array>
#include <span>

int main() {
    // libc++ hardened mode: lightweight checks for production

    // vector bounds
    std::vector<int> v = {1, 2, 3};
    // v[10];  // Aborts with "vector[] index out of bounds"

    // array bounds
    std::array<int, 3> a = {1, 2, 3};
    // a[5];   // Aborts

    // span bounds
    std::span<int> s(v);
    // s[10];  // Aborts

    std::cout << "Hardened mode catches:\n";
    std::cout << "  operator[], front(), back() out of bounds\n";
    std::cout << "  span subscript violations\n";
    std::cout << "  optional dereference without value\n";
    std::cout << "  string_view null construction\n";
    std::cout << "\nOverhead: ~1-5% (production-safe!)\n";

    // Evolution of libc++ hardening:
    // Clang 17: _LIBCPP_ENABLE_HARDENED_MODE=1
    // Clang 18: _LIBCPP_HARDENING_MODE = FAST/EXTENSIVE/DEBUG/NONE
}

```

### Q3: OOB access: crashes with hardening, silent without

```cpp

// Compile two ways:
//   g++ -std=c++20 -O2 oob.cpp -o oob_release
//   g++ -std=c++20 -D_GLIBCXX_DEBUG -O0 -g oob.cpp -o oob_debug

#include <iostream>
#include <vector>

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50};

    // Out-of-bounds write (buffer overflow)
    // Without hardening: silently corrupts adjacent memory
    // With _GLIBCXX_DEBUG: aborts with diagnostic

    // Simulate the dangerous access:
    std::cout << "Vector size: " << v.size() << '\n';
    std::cout << "Attempting v[100]...\n";

    // Uncomment to test:
    // int val = v[100];  // Release: reads garbage. Debug: ABORT.
    // v[100] = 999;      // Release: silent memory corruption! Debug: ABORT.

    // Why this matters for security:
    // 1. Read OOB (CWE-125): Information disclosure (Heartbleed-style)
    // 2. Write OOB (CWE-787): Code execution (stack/heap overflow exploit)
    // 3. Both are in Top 5 of CWE Top 25

    std::cout << "\nRelease (no hardening):\n";
    std::cout << "  v[100] -> reads garbage from heap (info leak)\n";
    std::cout << "  v[100]=x -> corrupts heap (potential RCE)\n";
    std::cout << "  Program continues running (DANGEROUS!)\n";

    std::cout << "\nDebug (_GLIBCXX_DEBUG):\n";
    std::cout << "  v[100] -> error: index 100 out of range\n";
    std::cout << "  Program aborts immediately (SAFE)\n";

    std::cout << "\nProduction (_GLIBCXX_ASSERTIONS):\n";
    std::cout << "  v[100] -> __glibcxx_assert fails -> abort\n";
    std::cout << "  ~5% overhead, no ABI change\n";
}

```

---

## Notes

- Complementary to #558 (production hardening + perf) and #735 (iterator validity).
- `_GLIBCXX_DEBUG` changes container layout (ABI break). ALL code must be compiled the same way.
- `_GLIBCXX_ASSERTIONS` is the lightweight production alternative (no ABI break).
- Google, Apple, and Microsoft all enable some form of STL hardening in production.
- Combine with AddressSanitizer for maximum coverage during testing.
