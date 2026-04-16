# Use reflection for enum-to-string conversion without macros

**Category:** Reflection (C++26)  
**Item:** #714  
**Standard:** C++17  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

`enumerators_of(^E)` returns all enumerators, and `identifier_of` gives their names. This replaces macros and external libraries for enum-to-string.

| Approach | LOC | Portable | Enum limits | Maintenance |
| --- | --- | --- | --- | --- |
| Manual switch | N cases | Yes | None | Per-change |
| X-macros | ~5 | Yes | None | Fragile |
| `magic_enum` | 0 | Limited | ~256 values | Zero |
| **C++26 reflection** | **~5** | **Yes** | **None** | **Zero** |

---

## Self-Assessment

### Q1: Iterate enum values and names with `enumerators_of`

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

enum class Weekday { Mon, Tue, Wed, Thu, Fri, Sat, Sun };

int main() {
    constexpr auto enumerators = std::meta::enumerators_of(^Weekday);
    static_assert(enumerators.size() == 7);

    std::cout << "Weekday enumerators:\n";
    template for (constexpr auto e : enumerators) {
        constexpr std::string_view name = std::meta::identifier_of(e);
        constexpr Weekday val = [:e:];
        std::cout << "  " << name << " = " << static_cast<int>(val) << '\n';
    }
    // Output:
    // Weekday enumerators:
    //   Mon = 0
    //   Tue = 1
    //   Wed = 2
    //   Thu = 3
    //   Fri = 4
    //   Sat = 5
    //   Sun = 6

    // Works with sparse enums too:
    enum class HttpCode { OK = 200, NotFound = 404, Error = 500 };
    template for (constexpr auto e : std::meta::enumerators_of(^HttpCode)) {
        std::cout << std::meta::identifier_of(e) << " = "
                  << static_cast<int>([:e:]) << '\n';
    }
}

```

### Q2: `constexpr to_string()` without macros

```cpp

#include <meta>
#include <iostream>
#include <string_view>

enum class Season { Spring, Summer, Autumn, Winter };
enum class Priority { Low = 1, Medium = 5, High = 10, Critical = 100 };

// Generic enum_to_string — works for ANY enum:
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) {
            return std::meta::identifier_of(e);
        }
    }
    return "<unknown>";
}

// Compile-time verification:
static_assert(to_string(Season::Spring) == "Spring");
static_assert(to_string(Season::Winter) == "Winter");
static_assert(to_string(Priority::Critical) == "Critical");

// Also: from_string:
template <typename E>
constexpr std::optional<E> from_string(std::string_view name) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (name == std::meta::identifier_of(e)) return [:e:];
    }
    return std::nullopt;
}

static_assert(from_string<Season>("Summer") == Season::Summer);

int main() {
    std::cout << to_string(Season::Autumn) << '\n';     // Autumn
    std::cout << to_string(Priority::High) << '\n';     // High

    auto s = from_string<Priority>("Medium");
    if (s) std::cout << "Parsed: " << static_cast<int>(*s) << '\n'; // 5
}

```

### Q3: Comparison with `magic_enum` library

```cpp

#include <meta>
#include <iostream>

enum class Color { Red, Green, Blue, Alpha };

// ===== magic_enum approach (C++17 library) =====
// #include <magic_enum.hpp>
// auto name = magic_enum::enum_name(Color::Red);    // "Red"
// auto val  = magic_enum::enum_cast<Color>("Blue"); // Color::Blue
//
// Limitations of magic_enum:
// 1. Default range: only [-128, 127] for enum values
//    enum class Big { X = 1000 }; // magic_enum CANNOT see this!
// 2. Relies on compiler-specific __PRETTY_FUNCTION__ / __FUNCSIG__
// 3. Increased compile time (instantiates for every value in range)
// 4. Not standard C++ — external dependency
// 5. Range must be customized per-enum for sparse/large enums

// ===== C++26 reflection approach =====
template <typename E>
constexpr std::string_view to_string(E value) {
    template for (constexpr auto e : std::meta::enumerators_of(^E)) {
        if (value == [:e:]) return std::meta::identifier_of(e);
    }
    return "<unknown>";
}

// Advantages of reflection:
// 1. No range limit — works for ALL enum values, any underlying type
// 2. Standard C++ — no external library needed
// 3. Faster compilation — iterates actual enumerators, not a range
// 4. Works with ALL compilers that support C++26
// 5. Zero runtime overhead (compile-time only)

enum class BigEnum : long long { X = 1'000'000, Y = 2'000'000 };
// magic_enum: FAILS (out of default range)
// reflection: WORKS
static_assert(to_string(BigEnum::X) == "X");

int main() {
    std::cout << to_string(Color::Green) << '\n';  // Green
    std::cout << to_string(BigEnum::Y) << '\n';    // Y
}

```

---

## Notes

- `enumerators_of` returns all explicitly declared enumerators (no synthetic values).
- Reflection-based conversion has no range limitations unlike `magic_enum`.
- `template for` expansion generates optimal code — equivalent to a hand-written switch.
- This is the canonical use case for C++26 reflection — often the first example shown.
- Combine `to_string` + `from_string` for complete bidirectional enum/string conversion.
