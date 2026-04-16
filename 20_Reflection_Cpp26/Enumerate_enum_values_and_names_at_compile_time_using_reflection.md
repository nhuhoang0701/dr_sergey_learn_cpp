# Enumerate enum values and names at compile time using reflection

**Category:** Reflection (C++26)  
**Item:** #536  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++26 reflection allows iterating over enum values at compile time using `std::meta::enumerators_of`. Combined with `identifier_of` and splicing, you can build enum-to-string and string-to-enum conversions without macros.

| API | Purpose |
| --- | --- |
| `^MyEnum` | Reflect the enum type |
| `enumerators_of(^E)` | Get a `vector<info>` of all enumerators |
| `identifier_of(e)` | Get enumerator name as `string_view` |
| `[:e:]` | Splice back to a value |
| `value_of<E>(e)` | Get the underlying value |

```cpp

Reflection flow:

  enum Color { Red, Green, Blue };

  ^Color ─→ meta::info (reflection of Color)
    │
  enumerators_of(^Color) ─→ [info(Red), info(Green), info(Blue)]
    │
  identifier_of(info(Red)) ─→ "Red"
  [:info(Red):] ─→ Color::Red (value)

```

---

## Self-Assessment

### Q1: Use `enumerators_of` to list all enumerator values

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

enum class Color { Red = 1, Green = 2, Blue = 4 };

int main() {
    // Get all enumerators:
    constexpr auto enumerators = std::meta::enumerators_of(^Color);

    // Iterate at compile time (template for + constexpr):
    std::cout << "Color enumerators:\n";
    template for (constexpr auto e : enumerators) {
        // identifier_of: get the name as string_view
        constexpr std::string_view name = std::meta::identifier_of(e);
        // [:e:] splices back to the actual value
        constexpr Color val = [:e:];
        std::cout << "  " << name << " = "
                  << static_cast<int>(val) << '\n';
    }
}
// Output:
//   Color enumerators:
//   Red = 1
//   Green = 2
//   Blue = 4

```

### Q2: Type-safe `enum_to_string()` without macros

```cpp

#include <meta>
#include <string_view>
#include <stdexcept>

enum class HttpStatus { OK = 200, NotFound = 404, ServerError = 500 };

// Generic enum_to_string using reflection:
template <typename E>
constexpr std::string_view enum_to_string(E value) {
    // Iterate enumerators at compile time:
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) {
            return std::meta::identifier_of(e);
        }
    }
    throw std::invalid_argument("Unknown enum value");
}

// Works for ANY enum type without changes:
static_assert(enum_to_string(HttpStatus::OK) == "OK");
static_assert(enum_to_string(HttpStatus::NotFound) == "NotFound");

enum class Direction { North, South, East, West };
static_assert(enum_to_string(Direction::East) == "East");

int main() {
    std::cout << enum_to_string(HttpStatus::ServerError) << '\n';
    // Output: ServerError

    // Comparison with old approach:
    // OLD: X-macro or manual switch with 20+ cases
    // NEW: 5 lines of generic code, works for ALL enums
}

```

### Q3: Compile-time `string_to_enum()` lookup table

```cpp

#include <meta>
#include <string_view>
#include <optional>
#include <array>
#include <algorithm>

enum class LogLevel { Debug, Info, Warning, Error, Fatal };

// Generic string_to_enum using reflection:
template <typename E>
constexpr std::optional<E> string_to_enum(std::string_view name) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (name == std::meta::identifier_of(e)) {
            return [:e:];
        }
    }
    return std::nullopt;
}

// Or generate a sorted lookup table for O(log N) search:
template <typename E>
constexpr auto make_enum_lookup_table() {
    constexpr auto enums = std::meta::enumerators_of(^E);
    std::array<std::pair<std::string_view, E>, enums.size()> table;

    std::size_t i = 0;
    template for (constexpr auto e : enums) {
        table[i++] = { std::meta::identifier_of(e), [:e:] };
    }

    std::ranges::sort(table, {}, &std::pair<std::string_view, E>::first);
    return table;
}

constexpr auto log_level_table = make_enum_lookup_table<LogLevel>();

int main() {
    // Linear search:
    auto level = string_to_enum<LogLevel>("Warning");
    if (level) {
        std::cout << "Parsed: " << enum_to_string(*level) << '\n';
    }

    // Table lookup (binary search):
    auto it = std::ranges::lower_bound(
        log_level_table, "Error",
        {}, &decltype(log_level_table)::value_type::first);
    if (it != log_level_table.end() && it->first == "Error") {
        std::cout << "Found: " << static_cast<int>(it->second) << '\n';
    }
}
// Output:
// Parsed: Warning
// Found: 3

```

---

## Notes

- `enumerators_of` works on both scoped (`enum class`) and unscoped (`enum`) enums.
- `template for` is the expansion statement for iterating compile-time ranges.
- All reflection happens at compile time — zero runtime overhead for the lookup tables.
- This replaces macro-heavy solutions like `BETTER_ENUMS`, `magic_enum`, and X-macros.
- `identifier_of` returns the source name exactly as written (case-sensitive).
