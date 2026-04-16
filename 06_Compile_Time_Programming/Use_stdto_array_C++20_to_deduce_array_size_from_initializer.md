# Use `std::to_array` (C++20) to Deduce Array Size from Initializer

**Category:** Compile-Time Programming  
**Item:** #206  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/container/array/to_array>  

---

## Topic Overview

### What Is `std::to_array`

`std::to_array` converts a C-style array or brace-enclosed initializer into a `std::array` with automatically deduced size and element type.

```cpp

auto arr = std::to_array({1, 2, 3});  // std::array<int, 3>
auto str = std::to_array("hello");    // std::array<char, 6> (includes '\0')

```

### Why Not Just Use `std::array<int, 3>{1, 2, 3}`

| Issue | `std::array` constructor | `std::to_array` |
| --- | --- | --- |
| Must specify size | `std::array<int, 3>{1,2,3}` | `std::to_array({1,2,3})` |
| Must specify type | `std::array<int, 3>{1,2,3}` | `std::to_array({1,2,3})` |
| Adding elements requires size change | Yes | No — auto-deduced |
| Works with C arrays | No direct conversion | `std::to_array(c_arr)` |
| Supports move semantics | Only brace init | Yes — moves from C array |

### Signature

```cpp

template<class T, std::size_t N>
constexpr std::array<std::remove_cv_t<T>, N> to_array(T (&a)[N]);      // copy

template<class T, std::size_t N>
constexpr std::array<std::remove_cv_t<T>, N> to_array(T (&&a)[N]);     // move

```

---

## Self-Assessment

### Q1: Replace `const int arr[] = {1,2,3}` with `std::to_array` and show the type is `std::array<int,3>`

```cpp

#include <iostream>
#include <array>
#include <type_traits>

int main() {
    // === OLD: C-style array ===
    const int old_arr[] = {1, 2, 3};
    // Type is const int[3] — no .size(), no iterators, decays to pointer

    // === NEW: std::to_array (C++20) ===
    auto arr = std::to_array({1, 2, 3});
    // Type is std::array<int, 3> — full container interface

    // Verify type at compile time
    static_assert(std::is_same_v<decltype(arr), std::array<int, 3>>);
    static_assert(arr.size() == 3);

    std::cout << "=== std::to_array basic usage ===\n";
    std::cout << "arr.size() = " << arr.size() << "\n";
    std::cout << "Elements: ";
    for (auto v : arr) std::cout << v << " ";
    std::cout << "\n";

    // === Deduces type from elements ===
    auto doubles = std::to_array({1.1, 2.2, 3.3, 4.4});
    static_assert(std::is_same_v<decltype(doubles), std::array<double, 4>>);
    std::cout << "\ndoubles.size() = " << doubles.size() << "\n";

    // === From C-style array ===
    int c_arr[] = {10, 20, 30, 40, 50};
    auto from_c = std::to_array(c_arr);
    static_assert(std::is_same_v<decltype(from_c), std::array<int, 5>>);
    std::cout << "from_c.size() = " << from_c.size() << "\n";

    // === String literal → array<char, N> ===
    auto chars = std::to_array("hello");
    static_assert(std::is_same_v<decltype(chars), std::array<char, 6>>); // includes '\0'
    std::cout << "\nchars from \"hello\": size=" << chars.size()
              << " (includes null terminator)\n";

    // === constexpr usage ===
    constexpr auto ct = std::to_array({100, 200, 300});
    static_assert(ct[0] == 100);
    static_assert(ct[2] == 300);
    std::cout << "\nconstexpr ct[1] = " << ct[1] << "\n";

    return 0;
}

```

**Expected output:**

```text

=== std::to_array basic usage ===
arr.size() = 3
Elements: 1 2 3

doubles.size() = 4
from_c.size() = 5

chars from "hello": size=6 (includes null terminator)

constexpr ct[1] = 200

```

### Q2: Show that `std::to_array` supports move for non-copyable element types

```cpp

#include <iostream>
#include <array>
#include <memory>
#include <string>

// === Non-copyable type ===
struct NoCopy {
    int value;
    NoCopy(int v) : value(v) { std::cout << "  NoCopy(" << v << ") constructed\n"; }
    NoCopy(NoCopy&& other) noexcept : value(other.value) {
        other.value = -1;
        std::cout << "  NoCopy(" << value << ") moved\n";
    }
    NoCopy(const NoCopy&) = delete;
    NoCopy& operator=(const NoCopy&) = delete;
    NoCopy& operator=(NoCopy&&) = default;
};

int main() {
    std::cout << "=== std::to_array with move-only types ===\n";

    // to_array MOVES from the temporary C-array
    // The rvalue-ref overload: to_array(T (&&a)[N])
    auto arr = std::to_array<NoCopy>({NoCopy{1}, NoCopy{2}, NoCopy{3}});

    std::cout << "\nResult: ";
    for (const auto& e : arr) std::cout << e.value << " ";
    std::cout << "\n";

    // === unique_ptr example ===
    std::cout << "\n=== std::to_array with unique_ptr ===\n";
    auto ptrs = std::to_array<std::unique_ptr<int>>({
        std::make_unique<int>(10),
        std::make_unique<int>(20),
        std::make_unique<int>(30)
    });

    std::cout << "ptrs.size() = " << ptrs.size() << "\n";
    for (const auto& p : ptrs) {
        std::cout << "*p = " << *p << "\n";
    }

    // === Why this matters ===
    std::cout << "\n=== Why this matters ===\n";
    std::cout << "Without to_array, you CANNOT create std::array<unique_ptr,N>\n";
    std::cout << "from an initializer list (initializer_list requires copy).\n";
    std::cout << "std::to_array's move overload solves this.\n";

    // This would NOT compile:
    // std::array<std::unique_ptr<int>, 2> bad = {
    //     std::make_unique<int>(1), std::make_unique<int>(2)
    // };
    // Error: std::array aggregate init copies elements on some implementations

    return 0;
}

```

**Expected output (move/construct order may vary):**

```text

=== std::to_array with move-only types ===
  NoCopy(1) constructed
  NoCopy(2) constructed
  NoCopy(3) constructed
  NoCopy(1) moved
  NoCopy(2) moved
  NoCopy(3) moved

Result: 1 2 3

=== std::to_array with unique_ptr ===
ptrs.size() = 3
*p = 10
*p = 20
*p = 30

=== Why this matters ===
Without to_array, you CANNOT create std::array<unique_ptr,N>
from an initializer list (initializer_list requires copy).
std::to_array's move overload solves this.

```

### Q3: Explain why `std::to_array` is preferred over brace-initializing `std::array` when element count is implicit

When the number of elements is determined by the initializer (not by the API contract), `std::to_array` is better:

```cpp

#include <iostream>
#include <array>
#include <type_traits>
#include <algorithm>
#include <numeric>

// === Problem: std::array requires explicit size ===
// If you add an element, you must update the size manually:
//   std::array<int, 4> data = {1, 2, 3, 4};
//   std::array<int, 5> data = {1, 2, 3, 4, 5}; // must change 4 → 5!

// === Solution: std::to_array deduces the size ===

// Compile-time configuration tables — size deduced from content
constexpr auto error_codes = std::to_array({
    200, 201, 204,
    301, 302,
    400, 401, 403, 404,
    500, 502, 503
});
// Adding a new code? Just add it — size updates automatically.

constexpr auto thresholds = std::to_array({0.1, 0.5, 0.9, 0.95, 0.99});

// === Comparison table ===
// std::array<int, 5> x = {1,2,3,4,5};    // must count manually → error-prone
// auto x = std::to_array({1,2,3,4,5});    // size deduced → less error-prone

// static_assert verifying size is correct without manual counting
static_assert(error_codes.size() == 12);
static_assert(thresholds.size() == 5);

// === When to still use explicit std::array<T, N> ===
// When size is part of the contract, not determined by content:
struct Color { float r, g, b, a; };
using Vec3 = std::array<float, 3>;  // Always 3 elements — conceptual meaning

int main() {
    std::cout << "=== std::to_array: Size Deduction ===\n";
    std::cout << "error_codes has " << error_codes.size() << " entries\n";
    std::cout << "thresholds has " << thresholds.size() << " entries\n";

    // Full container interface
    std::cout << "\nFirst error code: " << error_codes.front() << "\n";
    std::cout << "Last error code: " << error_codes.back() << "\n";

    bool has_404 = std::find(error_codes.begin(), error_codes.end(), 404)
                   != error_codes.end();
    std::cout << "Contains 404: " << (has_404 ? "yes" : "no") << "\n";

    // === constexpr array operations ===
    constexpr auto sorted_codes = [] {
        auto copy = error_codes;
        std::sort(copy.begin(), copy.end());
        return copy;
    }();

    std::cout << "\nSorted codes: ";
    for (auto c : sorted_codes) std::cout << c << " ";
    std::cout << "\n";

    // === Summary ===
    std::cout << "\n=== When to use std::to_array ===\n";
    std::cout << "✓ Configuration tables (size from content)\n";
    std::cout << "✓ Converting C arrays to std::array\n";
    std::cout << "✓ Move-only element types\n";
    std::cout << "✗ Fixed-size contracts (Vec3, Matrix4x4) — use std::array<T,N>\n";

    return 0;
}

```

**Expected output:**

```text

=== std::to_array: Size Deduction ===
error_codes has 12 entries
thresholds has 5 entries

First error code: 200
Last error code: 503
Contains 404: yes

Sorted codes: 200 201 204 301 302 400 401 403 404 500 502 503

=== When to use std::to_array ===
✓ Configuration tables (size from content)
✓ Converting C arrays to std::array
✓ Move-only element types
✗ Fixed-size contracts (Vec3, Matrix4x4) — use std::array<T,N>

```

---

## Notes

- `std::to_array` deduces both the element type and size from the initializer — no manual counting.
- The move overload `to_array(T(&&)[N])` enables creating `std::array` of move-only types.
- It's `constexpr` — works at compile time for building lookup tables.
- Use `std::to_array` when size is determined by content; use `std::array<T, N>` when `N` is a contract.
- `std::to_array("hello")` produces `std::array<char, 6>` — includes the null terminator.
- For string literals specifically, `std::string_view` is usually more appropriate than `std::to_array`.
