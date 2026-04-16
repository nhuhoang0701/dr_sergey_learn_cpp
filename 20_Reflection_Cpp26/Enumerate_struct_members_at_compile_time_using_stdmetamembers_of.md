# Enumerate struct members at compile time using std::meta::members_of

**Category:** Reflection (C++26)  
**Item:** #616  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

`std::meta::members_of` and related queries provide compile-time introspection of struct members, including names and types.

| Query | Returns |
| --- | --- |
| `members_of(^S)` | All members (data, static, functions) |
| `nonstatic_data_members_of(^S)` | Only instance data members |
| `identifier_of(m)` | Member name as `string_view` |
| `type_of(m)` | Member type as `meta::info` |
| `display_string_of(info)` | Human-readable type name |

---

## Self-Assessment

### Q1: Iterate non-static data members of a struct

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>

struct Config {
    std::string host;
    int port;
    bool verbose;
    double timeout;
};

int main() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^Config);

    std::cout << "Config has " << members.size() << " fields:\n";
    template for (constexpr auto m : members) {
        std::cout << "  " << std::meta::identifier_of(m) << '\n';
    }

    // Access on an instance:
    Config cfg{"localhost", 8080, true, 30.0};
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m) << " = ";
        // Use if constexpr to handle types that need different printing:
        using MemberType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MemberType, std::string>) {
            std::cout << '"' << cfg.[:m:] << '"';
        } else if constexpr (std::is_same_v<MemberType, bool>) {
            std::cout << (cfg.[:m:] ? "true" : "false");
        } else {
            std::cout << cfg.[:m:];
        }
        std::cout << '\n';
    }
}
// Output:
// Config has 4 fields:
//   host
//   port
//   verbose
//   timeout
// host = "localhost"
// port = 8080
// verbose = true
// timeout = 30

```

### Q2: Print name and type of each member

```cpp

#include <meta>
#include <iostream>
#include <string_view>

struct Particle {
    float x, y, z;
    float velocity;
    int id;
    bool active;
};

template <typename T>
void describe_struct() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    constexpr std::string_view type_name = std::meta::identifier_of(^T);

    std::cout << "struct " << type_name << " {\n";
    template for (constexpr auto m : members) {
        constexpr auto name = std::meta::identifier_of(m);
        constexpr auto type = std::meta::display_string_of(std::meta::type_of(m));
        constexpr auto offset = std::meta::offset_of(m);
        constexpr auto size = std::meta::size_of(std::meta::type_of(m));

        std::cout << "    " << type << " " << name
                  << ";  // offset=" << offset
                  << " size=" << size << '\n';
    }
    std::cout << "};  // total size=" << sizeof(T) << '\n';
}

int main() {
    describe_struct<Particle>();
}
// Output:
// struct Particle {
//     float x;  // offset=0 size=4
//     float y;  // offset=4 size=4
//     float z;  // offset=8 size=4
//     float velocity;  // offset=12 size=4
//     int id;  // offset=16 size=4
//     bool active;  // offset=20 size=1
// };  // total size=24

```

### Q3: Generic `to_string()` for any aggregate

```cpp

#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string generic_to_string(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;

    oss << std::meta::identifier_of(^T) << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << std::meta::identifier_of(m) << "=";

        using MemberType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MemberType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MemberType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else {
            oss << obj.[:m:];
        }
    }
    oss << "}";
    return oss.str();
}

struct Employee {
    std::string name;
    int id;
    double salary;
    bool active;
};

struct Vec2 { float x, y; };

int main() {
    Employee e{"Alice", 42, 95000.0, true};
    std::cout << generic_to_string(e) << '\n';
    // Output: Employee{name="Alice", id=42, salary=95000, active=true}

    Vec2 v{1.5f, 2.5f};
    std::cout << generic_to_string(v) << '\n';
    // Output: Vec2{x=1.5, y=2.5}
}

```

---

## Notes

- `nonstatic_data_members_of` is preferred over `members_of` for data-only iteration.
- `identifier_of(^T)` returns the type name itself.
- `display_string_of` gives a human-readable type string (implementation-defined format).
- `offset_of` and `size_of` enable layout introspection without macros.
- Reflection works with inheritance — `bases_of(^Derived)` gives base classes.
