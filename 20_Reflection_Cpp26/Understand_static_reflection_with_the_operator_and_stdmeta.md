# Understand static reflection with the ^ operator and std::meta

**Category:** Reflection (C++26)  
**Item:** #711  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

C++26 introduces **static reflection** via the `^` (reflect) operator and the `std::meta` namespace. `^T` produces a value of type `std::meta::info` that represents entity `T` at compile time.

| Concept | Description |
| --- | --- |
| `^entity` | Reflect operator — turns any entity into a `meta::info` value |
| `std::meta::info` | Opaque compile-time type representing a reflected entity |
| `identifier_of(info)` | Get the source name as `string_view` |
| `type_of(info)` | Get the type of a reflected member |
| `members_of(info)` | Get all members of a reflected class/struct |
| `[:info:]` | Splice — convert back from reflection to code |

```cpp

Reflection pipeline:

  Source code entity  ───^entity───→  std::meta::info
                                         │
                                    query APIs
                                    (name, type,
                                     members...)
                                         │
  Generated code      ──[:info:]──────┘

```

---

## Self-Assessment

### Q1: Use `^T` to obtain a reflection and query its name

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

struct Widget {
    int id;
    std::string name;
    double price;
};

int main() {
    // Reflect a type:
    constexpr std::meta::info widget_refl = ^Widget;

    // Query the name:
    constexpr std::string_view type_name = std::meta::identifier_of(widget_refl);
    std::cout << "Type: " << type_name << '\n';  // Type: Widget

    // Reflect built-in types:
    constexpr auto int_refl = ^int;
    std::cout << "Built-in: " << std::meta::display_string_of(int_refl) << '\n';
    // Built-in: int

    // Reflect a namespace:
    constexpr auto std_refl = ^std;
    std::cout << "Namespace: " << std::meta::identifier_of(std_refl) << '\n';
    // Namespace: std

    // Reflect an enum:
    enum Color { Red, Green, Blue };
    constexpr auto color_refl = ^Color;
    std::cout << "Enum: " << std::meta::identifier_of(color_refl) << '\n';
    // Enum: Color
}

```

### Q2: Enumerate data members of a struct using `members_of`

```cpp

#include <meta>
#include <iostream>

struct Config {
    std::string host;
    int port;
    bool ssl;
};

template <typename T>
void print_members() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);

    std::cout << std::meta::identifier_of(^T) << " has "
              << members.size() << " members:\n";

    template for (constexpr auto m : members) {
        std::cout << "  "
                  << std::meta::display_string_of(std::meta::type_of(m))
                  << " " << std::meta::identifier_of(m) << '\n';
    }
}

int main() {
    print_members<Config>();
    // Output:
    // Config has 3 members:
    //   std::string host
    //   int port
    //   bool ssl

    // Works for any struct:
    struct Vec3 { float x, y, z; };
    print_members<Vec3>();
    // Vec3 has 3 members:
    //   float x
    //   float y
    //   float z
}

```

### Q3: Reflections are compile-time values, not runtime objects

```cpp

#include <meta>
#include <iostream>
#include <type_traits>

// Key insight: std::meta::info is a COMPILE-TIME value.
// It exists only during compilation - zero runtime overhead.

// 1. You can pass reflections to consteval functions:
consteval bool has_member_named(std::meta::info type, std::string_view name) {
    for (auto m : std::meta::nonstatic_data_members_of(type)) {
        if (std::meta::identifier_of(m) == name) return true;
    }
    return false;
}

struct Player { std::string name; int hp; int level; };

// 2. Use in static_assert (purely compile-time):
static_assert(has_member_named(^Player, "hp"));
static_assert(!has_member_named(^Player, "mana"));

// 3. Use in constexpr variables:
constexpr auto player_refl = ^Player;
constexpr auto member_count = std::meta::nonstatic_data_members_of(player_refl).size();
static_assert(member_count == 3);

// 4. You CANNOT store meta::info at runtime:
// std::meta::info runtime_var;  // ERROR: not a runtime type
// std::vector<std::meta::info> v; // ERROR: cannot store in runtime container

// 5. consteval function — MUST be evaluated at compile time:
consteval std::size_t count_fields(std::meta::info t) {
    return std::meta::nonstatic_data_members_of(t).size();
}

int main() {
    // Compile-time constant derived from reflection:
    constexpr auto n = count_fields(^Player);
    std::cout << "Player has " << n << " fields\n";  // 3

    // The reflection itself has zero runtime footprint.
    // sizeof(std::meta::info) is NOT meaningful at runtime.
    static_assert(std::is_trivially_copyable_v<std::meta::info>);
}

```

---

## Notes

- `^` can reflect: types, namespaces, enums, variables, functions, templates.
- `std::meta::info` is an opaque, literal type — only meaningful at compile time.
- All `std::meta::` query functions are `consteval`.
- Reflections have zero runtime cost — they exist only during compilation.
- P2996R5 is the latest revision of the static reflection proposal.
