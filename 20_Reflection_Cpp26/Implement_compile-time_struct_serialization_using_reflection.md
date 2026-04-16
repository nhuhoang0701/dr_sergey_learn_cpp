# Implement compile-time struct serialization using reflection

**Category:** Reflection (C++26)  
**Item:** #619  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

Reflection enables writing generic serialization that works for any aggregate type without macros or manual registration.

| Approach | LOC per type | Maintenance | Type safety |
| --- | --- | --- | --- |
| Hand-written `toJson()` | ~10-30 | Manual per field | Compile-time |
| X-Macros | ~5 (macro) | Fragile | Limited |
| `BOOST_DESCRIBE` | ~3 (registration) | Per-type registration | Good |
| **Reflection** | **0** | **Zero — automatic** | **Full** |

---

## Self-Assessment

### Q1: `consteval` JSON-like serializer for any aggregate

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <string>
#include <sstream>
#include <iostream>

// Generic JSON serializer using reflection:
template <typename T>
std::string to_json(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;

        oss << '"' << std::meta::identifier_of(m) << "": ";

        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else if constexpr (std::is_arithmetic_v<MType>) {
            oss << obj.[:m:];
        } else {
            // Recursively serialize nested structs:
            oss << to_json(obj.[:m:]);
        }
    }
    oss << "}";
    return oss.str();
}

struct Address { std::string city; int zip; };
struct Person { std::string name; int age; Address addr; };

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << to_json(p) << '\n';
    // Output: {"name": "Alice", "age": 30, "addr": {"city": "NYC", "zip": 10001}}

    Address a{"London", 12345};
    std::cout << to_json(a) << '\n';
    // Output: {"city": "London", "zip": 12345}
}

```

### Q2: Binary serialization layout using `meta::offset_of` and `meta::size_of`

```cpp

#include <meta>
#include <cstring>
#include <vector>
#include <iostream>
#include <cstdint>

struct Packet {
    uint32_t id;
    uint16_t type;
    uint16_t flags;
    double payload;
};

// Compile-time layout descriptor:
template <typename T>
struct LayoutInfo {
    struct Field {
        std::string_view name;
        std::size_t offset;
        std::size_t size;
    };

    static constexpr auto fields = []{
        constexpr auto members = std::meta::nonstatic_data_members_of(^T);
        std::array<Field, members.size()> result{};
        std::size_t i = 0;
        template for (constexpr auto m : members) {
            result[i++] = {
                std::meta::identifier_of(m),
                std::meta::offset_of(m),
                std::meta::size_of(std::meta::type_of(m))
            };
        }
        return result;
    }();
};

// Binary serialize using layout info:
template <typename T>
std::vector<std::byte> serialize_binary(const T& obj) {
    std::vector<std::byte> buffer(sizeof(T));
    // Copy field by field using reflection-derived offsets:
    for (const auto& f : LayoutInfo<T>::fields) {
        std::memcpy(
            buffer.data() + f.offset,
            reinterpret_cast<const std::byte*>(&obj) + f.offset,
            f.size);
    }
    return buffer;
}

template <typename T>
T deserialize_binary(const std::vector<std::byte>& buffer) {
    T obj;
    std::memcpy(&obj, buffer.data(), sizeof(T));
    return obj;
}

int main() {
    Packet p{42, 1, 0x0F, 3.14};
    auto bytes = serialize_binary(p);
    auto p2 = deserialize_binary<Packet>(bytes);

    std::cout << "id=" << p2.id << " type=" << p2.type
              << " payload=" << p2.payload << '\n';
    // Output: id=42 type=1 payload=3.14

    // Print layout:
    for (const auto& f : LayoutInfo<Packet>::fields) {
        std::cout << f.name << ": offset=" << f.offset
                  << " size=" << f.size << '\n';
    }
}

```

### Q3: Reflection vs X-macros vs `BOOST_DESCRIBE`

```cpp

#include <meta>
#include <iostream>
#include <string>

// ========== OLD WAY 1: X-Macros ==========
// #define PERSON_FIELDS(X) X(std::string, name) X(int, age) X(double, height)
// struct PersonXMacro {
//     #define DECLARE(type, name) type name;
//     PERSON_FIELDS(DECLARE)
//     #undef DECLARE
// };
// void print_xmacro(const PersonXMacro& p) {
//     #define PRINT(type, name) std::cout << #name << "=" << p.name << '\n';
//     PERSON_FIELDS(PRINT)
//     #undef PRINT
// }
// Problems: fragile, no type safety, hard to debug, preprocessor pollution

// ========== OLD WAY 2: BOOST_DESCRIBE ==========
// struct PersonBoost { std::string name; int age; double height; };
// BOOST_DESCRIBE_STRUCT(PersonBoost, (), (name, age, height))
// // Must manually register each field name -- easy to forget one

// ========== NEW WAY: C++26 Reflection ==========
struct Person {
    std::string name;
    int age;
    double height;
};
// NOTHING ELSE NEEDED. No macros, no registration.

template <typename T>
void print_all_fields(const T& obj) {
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^T)) {
        std::cout << std::meta::identifier_of(m) << " = ";
        using MType = [:std::meta::type_of(m):];
        if constexpr (std::is_same_v<MType, std::string>)
            std::cout << '"' << obj.[:m:] << '"';
        else
            std::cout << obj.[:m:];
        std::cout << '\n';
    }
}

int main() {
    Person p{"Alice", 30, 5.6};
    print_all_fields(p);  // Works for ANY struct, zero registration
    // Output:
    // name = "Alice"
    // age = 30
    // height = 5.6

    // Add a new field to Person -> print_all_fields automatically picks it up.
    // With X-macros or BOOST_DESCRIBE: must also update the macro/registration.
}

```

---

## Notes

- `meta::offset_of(m)` returns the byte offset of a member within its struct.
- `meta::size_of(info)` returns the size in bytes of a reflected type.
- Reflection serialization is inherently type-safe — type mismatches are compile errors.
- For binary serialization, always consider endianness and alignment.
- Nested struct serialization requires recursive `to_json` calls.
