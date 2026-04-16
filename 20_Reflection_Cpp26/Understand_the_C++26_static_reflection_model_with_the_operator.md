# Understand the C++26 static reflection model with the ^ operator

**Category:** Reflection (C++26)  
**Item:** #615  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

The `^` (caret / reflect) operator is the entry point to C++26 static reflection. It converts compile-time entities into values of type `std::meta::info`.

| Operand | Example | Reflects |
| --- | --- | --- |
| Type | `^int`, `^MyClass` | The type itself |
| Variable | `^myVar` | The variable declaration |
| Function | `^myFunc` | The function |
| Namespace | `^std` | The namespace |
| Enum | `^Color` | The enum type |
| Enumerator | `^Color::Red` | Individual enumerator |

---

## Self-Assessment

### Q1: Use `^int` to obtain a `std::meta::info` value

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>

int main() {
    // ^ operator produces a std::meta::info:
    constexpr std::meta::info int_info = ^int;
    constexpr std::meta::info double_info = ^double;
    constexpr std::meta::info string_info = ^std::string;

    // Query the display name:
    std::cout << std::meta::display_string_of(int_info) << '\n';    // int
    std::cout << std::meta::display_string_of(double_info) << '\n';  // double
    std::cout << std::meta::display_string_of(string_info) << '\n';  // std::string

    // Reflect user-defined types:
    struct Point { int x, y; };
    constexpr auto point_info = ^Point;
    std::cout << std::meta::identifier_of(point_info) << '\n';  // Point

    // Reflect a template:
    constexpr auto vec_info = ^std::vector;
    std::cout << std::meta::display_string_of(vec_info) << '\n';
    // std::vector
}

```

### Q2: Difference between a reflection (`meta::info`) and a type

```cpp

#include <meta>
#include <iostream>
#include <type_traits>

// A reflection is NOT a type. It's a compile-time VALUE.
//
//   int            → a type (used in declarations, sizeof, etc.)
//   ^int           → a meta::info value (used in consteval queries)
//   [:^int:]       → spliced back into a type

int main() {
    // ^int is a VALUE, not a type:
    constexpr auto r = ^int;           // OK: value
    // int x = r;                     // ERROR: r is meta::info, not int

    // You CANNOT use meta::info where a type is expected:
    // sizeof(^int);      // ERROR: ^int is a value, not a type
    // std::vector<^int>;  // ERROR

    // You CAN splice it back to a type:
    using T = [:r:];        // T is int
    static_assert(std::is_same_v<T, int>);

    T value = 42;           // int value = 42;
    std::cout << value << '\n';

    // Comparison: meta::info values compare by identity:
    static_assert(^int == ^int);           // same type -> equal
    static_assert(^int != ^double);        // different type -> not equal
    static_assert(^int != ^unsigned int);  // int != unsigned int

    // meta::info is a literal type:
    constexpr auto a = ^int;
    constexpr auto b = ^int;
    static_assert(a == b);  // OK, constexpr comparison
}

```

### Q3: Reflections as constant expressions passed to `consteval` functions

```cpp

#include <meta>
#include <iostream>

// consteval: MUST be evaluated at compile time
consteval std::size_t field_count(std::meta::info type) {
    return std::meta::nonstatic_data_members_of(type).size();
}

consteval bool is_empty_struct(std::meta::info type) {
    return std::meta::nonstatic_data_members_of(type).empty();
}

consteval bool has_field(std::meta::info type, std::string_view name) {
    for (auto m : std::meta::nonstatic_data_members_of(type)) {
        if (std::meta::identifier_of(m) == name) return true;
    }
    return false;
}

struct Empty {};
struct Pair { int first; int second; };
struct Config { std::string host; int port; bool debug; };

// All evaluated at compile time:
static_assert(field_count(^Empty) == 0);
static_assert(field_count(^Pair) == 2);
static_assert(field_count(^Config) == 3);

static_assert(is_empty_struct(^Empty));
static_assert(!is_empty_struct(^Pair));

static_assert(has_field(^Config, "port"));
static_assert(!has_field(^Config, "timeout"));

int main() {
    // Use compile-time reflection results at runtime:
    constexpr auto n = field_count(^Config);
    std::cout << "Config fields: " << n << '\n';   // 3

    // Conditional compilation based on reflection:
    if constexpr (has_field(^Config, "debug")) {
        std::cout << "Config has debug field\n";
    }
}

```

---

## Notes

- `^` is a unary prefix operator with highest precedence.
- `^` can be applied to any named entity visible at the point of reflection.
- `std::meta::info` is a special type: literal, trivially copyable, but only meaningful at compile time.
- Reflections cannot be stored in runtime containers or passed across translation units.
- `consteval` functions accepting `meta::info` enable powerful compile-time metaprogramming.
