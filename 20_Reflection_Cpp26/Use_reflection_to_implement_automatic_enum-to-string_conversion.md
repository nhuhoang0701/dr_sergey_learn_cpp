# Use reflection to implement automatic enum-to-string conversion

**Category:** Reflection (C++26)  
**Item:** #618  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

Automatic enum-to-string is the quintessential reflection use case. With `enumerators_of` and `identifier_of`, a single generic function replaces all manual switch statements.

```cpp

Before reflection:         After reflection:

switch(color) {            template for (auto e : enumerators_of(^E))
  case Red:   return "Red";    if (value == [:e:])
  case Green: return "Green";      return identifier_of(e);
  case Blue:  return "Blue";
  // Must update for every new color!
}                          // Automatically handles ALL enumerators

```

---

## Self-Assessment

### Q1: Iterate all enumerators with `enumerators_of(^MyEnum)`

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

enum class Fruit { Apple, Banana, Cherry, Date, Elderberry };

int main() {
    constexpr auto enumerators = std::meta::enumerators_of(^Fruit);

    // Print all enumerator names:
    std::cout << "Fruit has " << enumerators.size() << " values:\n";
    template for (constexpr auto e : enumerators) {
        std::cout << "  " << std::meta::identifier_of(e)
                  << " (" << static_cast<int>([:e:]) << ")\n";
    }
    // Output:
    // Fruit has 5 values:
    //   Apple (0)
    //   Banana (1)
    //   Cherry (2)
    //   Date (3)
    //   Elderberry (4)

    // Count enumerators at compile time:
    static_assert(enumerators.size() == 5);

    // Get first and last names:
    constexpr auto first = std::meta::identifier_of(enumerators.front());
    constexpr auto last  = std::meta::identifier_of(enumerators.back());
    static_assert(first == "Apple");
    static_assert(last == "Elderberry");
}

```

### Q2: Generic `to_string()` without macros

```cpp

#include <meta>
#include <iostream>
#include <string_view>

enum class LogLevel   { Trace, Debug, Info, Warn, Error, Fatal };
enum class Permission { Read = 1, Write = 2, Execute = 4, Admin = 8 };

// ONE function for ALL enums:
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) return std::meta::identifier_of(e);
    }
    return "<unknown>";
}

// Reverse: string to enum:
template <typename E>
constexpr std::optional<E> parse_enum(std::string_view name) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (name == std::meta::identifier_of(e)) return [:e:];
    }
    return std::nullopt;
}

// Compile-time proof:
static_assert(to_string(LogLevel::Info) == "Info");
static_assert(to_string(Permission::Execute) == "Execute");

int main() {
    // Works for any enum:
    std::cout << to_string(LogLevel::Fatal) << '\n';       // Fatal
    std::cout << to_string(Permission::Admin) << '\n';     // Admin

    // Parse back:
    auto lvl = parse_enum<LogLevel>("Warn");
    if (lvl) std::cout << "Parsed: " << to_string(*lvl) << '\n'; // Warn

    // Adding a new LogLevel value like `Panic` requires:
    // - Old way: update switch statement (easy to forget)
    // - New way: NOTHING. to_string() handles it automatically.
}

```

### Q3: Compare reflection vs `magic_enum` library

```cpp

#include <meta>
#include <iostream>
// #include <magic_enum.hpp>  // for comparison

enum class Color { Red, Green, Blue };

// ===== magic_enum =====
// auto name = magic_enum::enum_name(Color::Red);     // "Red"
// auto val  = magic_enum::enum_cast<Color>("Green"); // Color::Green
//
// magic_enum internals:
// - Uses __PRETTY_FUNCTION__ / __FUNCSIG__ compiler extension
// - Instantiates a template for EACH value in range [-128, 127]
// - Checks if the resulting string contains a valid identifier
// - ~256 template instantiations per enum type!
// - Fails for values outside the range
// - Not portable across all compilers

// ===== C++26 reflection =====
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) return std::meta::identifier_of(e);
    }
    return "<unknown>";
}

// reflection internals:
// - Compiler directly queries its own symbol table
// - Only iterates ACTUAL enumerators (not a range)
// - No template instantiation trick needed
// - Works for ANY enum value, any underlying type
// - Standard C++, fully portable

// Practical differences:
enum class LargeEnum : uint64_t {
    A = 0, B = 1'000'000, C = 2'000'000'000'000
};

// magic_enum: CANNOT handle values > 128 without custom range
// reflection: works perfectly:
static_assert(to_string(LargeEnum::C) == "C");

int main() {
    std::cout << to_string(LargeEnum::B) << '\n'; // B
    // magic_enum: undefined behavior or empty string
    // reflection: "B" (correct)
}

```

---

## Notes

- This file focuses on the practical implementation; the file on `enum-to-string without macros` covers the `magic_enum` comparison in more depth.
- `to_string` + `parse_enum` together provide complete bidirectional conversion.
- The reflection approach compiles faster than `magic_enum` for large enums.
- No external dependencies required — pure standard C++26.
- Both scoped (`enum class`) and unscoped (`enum`) enums are supported.
