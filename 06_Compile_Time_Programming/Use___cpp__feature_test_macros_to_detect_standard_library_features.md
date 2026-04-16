# Use `__cpp_*` Feature Test Macros to Detect Standard Library Features

**Category:** Compile-Time Programming  
**Item:** #160  
**Standard:** C++20 (`<version>` header), macros defined incrementally since C++11  
**Reference:** <https://en.cppreference.com/w/cpp/feature_test>  

---

## Topic Overview

### What Are Feature Test Macros

Feature test macros are predefined preprocessor macros that let you detect at compile time whether a specific language or library feature is available. They follow naming conventions:

| Pattern | Scope | Example |
| --- | --- | --- |
| `__cpp_*` | Language features | `__cpp_concepts`, `__cpp_constexpr` |
| `__cpp_lib_*` | Library features | `__cpp_lib_ranges`, `__cpp_lib_format` |
| `__has_include(<header>)` | Header availability | `__has_include(<format>)` |

### Why Use Them

| Approach | Problem |
| --- | --- |
| Check `__cplusplus >= 202002L` | Compiler may claim C++20 but not implement all features |
| Check compiler version | Different compilers implement features at different versions |
| Feature test macros | Directly tests if the specific feature is available |

### Key Macros

| Macro | Feature | Value |
| --- | --- | --- |
| `__cpp_lib_ranges` | `<ranges>` | `≥ 201911L` |
| `__cpp_lib_format` | `<format>` | `≥ 202110L` |
| `__cpp_lib_expected` | `<expected>` | `≥ 202202L` |
| `__cpp_concepts` | `concept` keyword | `≥ 202002L` |
| `__cpp_constexpr` | Expanded constexpr | Value increases per standard |
| `__cpp_lib_coroutine` | `<coroutine>` | `≥ 201902L` |
| `__cpp_lib_jthread` | `std::jthread` | `≥ 201911L` |

### The `<version>` Header (C++20)

Include `<version>` to get all `__cpp_lib_*` macros without including the actual feature headers.

---

## Self-Assessment

### Q1: Use `#if __cpp_lib_ranges` to conditionally enable range-based code

```cpp

#include <iostream>
#include <vector>
#include <algorithm>

// <version> provides all __cpp_lib_* macros (C++20)
#if __has_include(<version>)
  #include <version>
#endif

// === Conditionally include ranges ===
#if defined(__cpp_lib_ranges) && __cpp_lib_ranges >= 201911L
  #include <ranges>
  #define HAS_RANGES 1
#else
  #define HAS_RANGES 0
#endif

// === Conditionally include format ===
#if defined(__cpp_lib_format) && __cpp_lib_format >= 202110L
  #include <format>
  #define HAS_FORMAT 1
#else
  #define HAS_FORMAT 0
#endif

// === Generic function with fallback ===
void print_sorted_evens(const std::vector<int>& data) {
#if HAS_RANGES
    std::cout << "Using std::ranges (available):\n  ";
    auto evens = data
        | std::views::filter([](int x) { return x % 2 == 0; });
    std::vector<int> sorted(evens.begin(), evens.end());
    std::ranges::sort(sorted);
    for (int v : sorted) std::cout << v << " ";
    std::cout << "\n";
#else
    std::cout << "Using pre-C++20 fallback:\n  ";
    std::vector<int> sorted;
    for (int v : data)
        if (v % 2 == 0) sorted.push_back(v);
    std::sort(sorted.begin(), sorted.end());
    for (int v : sorted) std::cout << v << " ";
    std::cout << "\n";
#endif
}

int main() {
    // === Report feature availability ===
    std::cout << "=== Feature Detection Report ===\n";

#ifdef __cpp_lib_ranges
    std::cout << "__cpp_lib_ranges = " << __cpp_lib_ranges << "\n";
#else
    std::cout << "__cpp_lib_ranges: NOT available\n";
#endif

#ifdef __cpp_lib_format
    std::cout << "__cpp_lib_format = " << __cpp_lib_format << "\n";
#else
    std::cout << "__cpp_lib_format: NOT available\n";
#endif

#ifdef __cpp_concepts
    std::cout << "__cpp_concepts = " << __cpp_concepts << "\n";
#else
    std::cout << "__cpp_concepts: NOT available\n";
#endif

#ifdef __cpp_constexpr
    std::cout << "__cpp_constexpr = " << __cpp_constexpr << "\n";
#else
    std::cout << "__cpp_constexpr: NOT available\n";
#endif

    // === Use the conditional function ===
    std::cout << "\n=== Conditional Code Path ===\n";
    std::vector<int> data = {9, 4, 7, 2, 8, 1, 6, 3, 5};
    print_sorted_evens(data);

    return 0;
}

```

**Expected output (C++20 compiler with ranges support):**

```text

=== Feature Detection Report ===
__cpp_lib_ranges = 201911
__cpp_lib_format = 202110
__cpp_concepts = 202002
__cpp_constexpr = 202207

=== Conditional Code Path ===
Using std::ranges (available):
  2 4 6 8

```

### Q2: Explain why feature test macros are more reliable than checking `__cplusplus` versions

Feature test macros directly test the specific feature you need, not the overall standard version:

| Scenario | `__cplusplus` | Feature test macro |
| --- | --- | --- |
| GCC 10 with `-std=c++20` but no `<format>` | `__cplusplus == 202002L` ✓ | `__cpp_lib_format` undefined ✗ |
| MSVC with partial C++23 | `__cplusplus == 202302L` ✓ | Each feature checked individually |
| Clang with libc++ vs libstdc++ | Same `__cplusplus` | Different `__cpp_lib_*` values |

```cpp

#include <iostream>

#if __has_include(<version>)
  #include <version>
#endif

int main() {
    std::cout << "=== Why __cplusplus alone is insufficient ===\n\n";

    // __cplusplus tells you the LANGUAGE standard mode
    std::cout << "__cplusplus = " << __cplusplus << "\n";

#if __cplusplus >= 202002L
    std::cout << "Compiler claims C++20 mode\n";
#endif

    // But this does NOT mean all C++20 library features exist!
    // The standard library and compiler frontend are separate.

    std::cout << "\n=== Feature-by-feature check is precise ===\n";

    // Each feature is independently testable
#ifdef __cpp_lib_ranges
    std::cout << "ranges:       YES (value=" << __cpp_lib_ranges << ")\n";
#else
    std::cout << "ranges:       NO\n";
#endif

#ifdef __cpp_lib_format
    std::cout << "format:       YES (value=" << __cpp_lib_format << ")\n";
#else
    std::cout << "format:       NO\n";
#endif

#ifdef __cpp_lib_jthread
    std::cout << "jthread:      YES (value=" << __cpp_lib_jthread << ")\n";
#else
    std::cout << "jthread:      NO\n";
#endif

#ifdef __cpp_lib_expected
    std::cout << "expected:     YES (value=" << __cpp_lib_expected << ")\n";
#else
    std::cout << "expected:     NO\n";
#endif

#ifdef __cpp_lib_coroutine
    std::cout << "coroutine:    YES (value=" << __cpp_lib_coroutine << ")\n";
#else
    std::cout << "coroutine:    NO\n";
#endif

    std::cout << "\n=== Key Insight ===\n";
    std::cout << "__cplusplus = standard mode (coarse)\n";
    std::cout << "__cpp_lib_* = individual feature (precise)\n";
    std::cout << "Macro values increase as features get defect fixes / extensions.\n";

    return 0;
}

```

### Q3: Write a header that provides a polyfill when a C++23 feature is absent

```cpp

// === my_expected.h — polyfill header for std::expected ===
#ifndef MY_EXPECTED_H
#define MY_EXPECTED_H

#if __has_include(<version>)
  #include <version>
#endif

#if defined(__cpp_lib_expected) && __cpp_lib_expected >= 202202L
  // === C++23 std::expected is available ===
  #include <expected>

  namespace my {
      template<typename T, typename E>
      using expected = std::expected<T, E>;

      template<typename E>
      using unexpected = std::unexpected<E>;
  }

#else
  // === Polyfill for compilers without std::expected ===
  #include <variant>
  #include <stdexcept>

  namespace my {
      template<typename E>
      struct unexpected {
          E error_;
          explicit unexpected(E e) : error_(std::move(e)) {}
          const E& error() const& { return error_; }
          E& error() & { return error_; }
      };

      template<typename T, typename E>
      class expected {
          std::variant<T, unexpected<E>> data_;
      public:
          expected(T val) : data_(std::move(val)) {}
          expected(unexpected<E> err) : data_(std::move(err)) {}

          bool has_value() const { return data_.index() == 0; }
          explicit operator bool() const { return has_value(); }

          T& value() & {
              if (!has_value()) throw std::runtime_error("bad expected access");
              return std::get<0>(data_);
          }
          const T& value() const& {
              if (!has_value()) throw std::runtime_error("bad expected access");
              return std::get<0>(data_);
          }
          const E& error() const& {
              return std::get<1>(data_).error();
          }

          T& operator*() & { return std::get<0>(data_); }
          const T& operator*() const& { return std::get<0>(data_); }
      };
  }

#endif
#endif // MY_EXPECTED_H

```

```cpp

// === main.cpp — uses the polyfill header ===
#include <iostream>
#include <string>

// In real code: #include "my_expected.h"
// For this self-contained example, the header content is above.

// === Example usage that works with both real and polyfill expected ===
my::expected<int, std::string> parse_int(const std::string& s) {
    try {
        std::size_t pos;
        int result = std::stoi(s, &pos);
        if (pos != s.size())
            return my::unexpected<std::string>("trailing characters");
        return result;
    } catch (const std::exception& e) {
        return my::unexpected<std::string>(e.what());
    }
}

int main() {
    std::cout << "=== Polyfill: my::expected ===\n";

#if defined(__cpp_lib_expected) && __cpp_lib_expected >= 202202L
    std::cout << "Using std::expected (C++23 native)\n";
#else
    std::cout << "Using polyfill (std::variant-based)\n";
#endif

    auto r1 = parse_int("42");
    if (r1) {
        std::cout << "parse_int(\"42\") = " << *r1 << "\n";
    }

    auto r2 = parse_int("abc");
    if (!r2) {
        std::cout << "parse_int(\"abc\") error: " << r2.error() << "\n";
    }

    auto r3 = parse_int("123xyz");
    if (!r3) {
        std::cout << "parse_int(\"123xyz\") error: " << r3.error() << "\n";
    }

    std::cout << "\n=== Polyfill Pattern ===\n";
    std::cout << "1. Check __cpp_lib_* macro\n";
    std::cout << "2. If available: alias the standard type\n";
    std::cout << "3. If absent: provide a minimal compatible implementation\n";
    std::cout << "4. User code uses your namespace (my::expected)\n";
    std::cout << "5. When all compilers catch up, remove the polyfill\n";

    return 0;
}

```

**Expected output:**

```text

=== Polyfill: my::expected ===
Using std::expected (C++23 native)
parse_int("42") = 42
parse_int("abc") error: stoi
parse_int("123xyz") error: trailing characters

=== Polyfill Pattern ===

1. Check __cpp_lib_* macro
2. If available: alias the standard type
3. If absent: provide a minimal compatible implementation
4. User code uses your namespace (my::expected)
5. When all compilers catch up, remove the polyfill

```

---

## Notes

- Include `<version>` (C++20) to get all `__cpp_lib_*` macros without pulling in feature headers.
- Use `__has_include(<header>)` to check header existence before `#include`.
- Feature test macro values are dates (e.g., `201911L`) — they increase when defect reports are applied.
- Always use `#if defined(MACRO) && MACRO >= VALUE` — not just `#ifdef` — to check minimum version.
- Language features use `__cpp_*`; library features use `__cpp_lib_*`.
- The polyfill pattern: check macro → if available use standard → else provide fallback.
