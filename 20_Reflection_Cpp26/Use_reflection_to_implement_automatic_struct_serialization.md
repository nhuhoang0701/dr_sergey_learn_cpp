# Use reflection to implement automatic struct serialization

**Category:** Reflection (C++26)  
**Item:** #712  
**Standard:** C++11  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2996r5.html>  

---

## Topic Overview

Reflection makes struct serialization fully automatic. Adding or removing a field from a struct automatically updates ALL serialization/deserialization code.

| Feature | Macro-based | Reflection |
| --- | --- | --- |
| Per-type registration | Required | None |
| New field handling | Manual update | Automatic |
| Nested types | Complex macros | Simple recursion |
| Type safety | Limited | Full |
| IDE support | Poor (macros) | Full |

---

## Self-Assessment

### Q1: Iterate fields with `members_of` and generate a serializer

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << std::meta::identifier_of(^T) << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << std::meta::identifier_of(m) << "=";

        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            oss << '"' << obj.[:m:] << '"';
        else if constexpr (std::is_same_v<MType, bool>)
            oss << (obj.[:m:] ? "true" : "false");
        else
            oss << obj.[:m:];
    }
    oss << "}";
    return oss.str();
}

struct Config {
    std::string host;
    int port;
    bool debug;
    double timeout;
};

int main() {
    Config cfg{"localhost", 8080, true, 30.0};
    std::cout << serialize(cfg) << '\n';
    // Config{host="localhost", port=8080, debug=true, timeout=30}

    struct Vec3 { float x, y, z; };
    Vec3 v{1.0f, 2.0f, 3.0f};
    std::cout << serialize(v) << '\n';
    // Vec3{x=1, y=2, z=3}
}

```

### Q2: Adding a new field automatically updates serialization

```cpp

#include <meta>
#include <iostream>
#include <string>
#include <sstream>

template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;
    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;
        oss << '"' << std::meta::identifier_of(m) << "": ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            oss << '"' << obj.[:m:] << '"';
        else
            oss << obj.[:m:];
    }
    oss << "}";
    return oss.str();
}

// Version 1:
struct UserV1 {
    std::string name;
    int age;
};

// Version 2 (added email and active):
struct UserV2 {
    std::string name;
    int age;
    std::string email;   // NEW
    bool active;          // NEW
};

int main() {
    UserV1 u1{"Alice", 30};
    std::cout << serialize(u1) << '\n';
    // {"name": "Alice", "age": 30}

    UserV2 u2{"Bob", 25, "bob@example.com", true};
    std::cout << serialize(u2) << '\n';
    // {"name": "Bob", "age": 25, "email": "bob@example.com", "active": 1}

    // The serialize() function is EXACTLY THE SAME for both.
    // Adding `email` and `active` requires ZERO changes to serialization code.

    // With manual approach, you'd need to update:
    // - toJson() method
    // - fromJson() method
    // - operator== (if applicable)
    // - hash function (if applicable)
    // - print/debug functions
    // = 5+ places to update per field change!
}

```

### Q3: Reflection vs macro-based approaches (Boost.Hana, REFLECT)

```cpp

#include <meta>
#include <iostream>
#include <string>

// ===== Boost.Hana approach =====
// BOOST_HANA_DEFINE_STRUCT(Person,
//     (std::string, name),
//     (int, age)
// );
// // Intrusive: modifies struct definition with macro
// // Fields must be declared INSIDE the macro
// // IDE cannot parse the struct normally

// ===== REFLECT macro approach =====
// struct Person { std::string name; int age; };
// REFLECT(Person, name, age)  // separate registration
// // Must list every field name
// // Easy to miss a field or get order wrong
// // Macro runs at preprocessing time - no type checking

// ===== Boost.Describe approach =====
// struct Person { std::string name; int age; };
// BOOST_DESCRIBE_STRUCT(Person, (), (name, age))
// // Non-intrusive but still manual
// // Must update when fields change

// ===== C++26 Reflection =====
struct Person {
    std::string name;
    int age;
    double height;
};
// No macros. No registration. Just the struct.

template <typename T>
void print_serialized(const T& obj) {
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        std::cout << std::meta::identifier_of(m) << ": ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            std::cout << obj.[:m:];
        else
            std::cout << obj.[:m:];
        std::cout << '\n';
    }
}

int main() {
    Person p{"Alice", 30, 5.6};
    print_serialized(p);
    // name: Alice
    // age: 30
    // height: 5.6

    // Comparison summary:
    // Boost.Hana:    intrusive macros, C++14, good metaprogramming
    // REFLECT macro: fragile, error-prone, preprocessor-level
    // Boost.Describe: non-intrusive, but manual registration
    // C++26 Reflection: zero registration, standard, type-safe
}

```

---

## Notes

- Reflection-based serialization requires zero per-type setup.
- Adding/removing fields is automatically reflected in all generic code.
- This eliminates entire categories of bugs (forgotten fields in serialization).
- For deserialization, the same pattern works in reverse with `obj.[:m:] = parsed_value`.
- Combine with `type_of` dispatch for format-specific serialization (JSON, XML, binary).
