# Implement compile-time-checked serialization with reflection (C++26)

**Category:** Serialization & Data Formats  
**Standard:** C++26  
**Reference:** <https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2023/p2996r0.html>  

---

## Topic Overview

C++26 static reflection (`^` operator and `std::meta` namespace) enables automatic serialization by enumerating struct members at compile time — no macros, no code generation, no boilerplate.

### Current State (Pre-Reflection): Macros

```cpp

// Pre-C++26: macros or code generators
#define SERIALIZABLE_FIELDS(Type, ...) \
    template<> struct Serializer<Type> { \
        static json to_json(const Type& obj) { /* ... */ } \
    };

// Boilerplate grows linearly with the number of types.

```

### C++26 Reflection Approach

```cpp

#include <meta>
#include <string>
#include <iostream>
#include <sstream>

// Automatic JSON serializer using C++26 reflection
template<typename T>
std::string to_json(const T& obj) {
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    // Iterate over all non-static data members at compile time
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            if (!first) oss << ",";
            first = false;

            // Get member name as string
            oss << "\"" << std::meta::identifier_of(member) << "\":";

            // Access the member value
            const auto& value = obj.[:member:];

            // Type-based formatting
            if constexpr (std::is_same_v<std::remove_cvref_t<decltype(value)>, std::string>) {
                oss << "\"" << value << "\"";
            } else if constexpr (std::is_same_v<std::remove_cvref_t<decltype(value)>, bool>) {
                oss << (value ? "true" : "false");
            } else {
                oss << value;
            }
        }
    }
    oss << "}";
    return oss.str();
}

// Usage — zero boilerplate per type!
struct Employee {
    std::string name;
    int age;
    double salary;
    bool active;
};

int main() {
    Employee emp{"Alice", 30, 85000.0, true};
    std::cout << to_json(emp) << "\n";
    // {"name":"Alice","age":30,"salary":85000,"active":true}
}

```

### Compile-Time Validation

```cpp

// Reflection enables compile-time checks on serializable types
template<typename T>
consteval bool is_serializable() {
    for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            using MemberType = typename[:std::meta::type_of(member):];
            if constexpr (!std::is_arithmetic_v<MemberType> &&
                          !std::is_same_v<MemberType, std::string>) {
                return false;  // Unsupported member type
            }
        }
    }
    return true;
}

static_assert(is_serializable<Employee>(), "Employee must be serializable");

```

---

## Self-Assessment

### Q1: Write a reflection-based deserializer from JSON

```cpp

// Conceptual C++26 code
template<typename T>
T from_json(const json_object& j) {
    T obj{};
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            constexpr auto name = std::meta::identifier_of(member);
            using MemberType = typename[:std::meta::type_of(member):];

            if (j.contains(name)) {
                obj.[:member:] = j[name].template get<MemberType>();
            }
            // Missing fields: keep default-initialized value
        }
    }
    return obj;
}

```

### Q2: Generate a schema description from a struct using reflection

```cpp

template<typename T>
void print_schema() {
    std::cout << "Schema for " << std::meta::identifier_of(^T) << ":\n";
    template for (constexpr auto member : std::meta::members_of(^T)) {
        if constexpr (std::meta::is_nonstatic_data_member(member)) {
            std::cout << "  " << std::meta::identifier_of(member)
                      << " : " << std::meta::identifier_of(std::meta::type_of(member))
                      << "\n";
        }
    }
}
// print_schema<Employee>();
// Schema for Employee:
//   name : string
//   age : int
//   salary : double
//   active : bool

```

### Q3: Why is reflection-based serialization safer than macro-based approaches

1. **Type-checked at compile time**: the splice operator `[:member:]` verifies the member exists
2. **Refactoring-safe**: renaming a field automatically updates the serialized name
3. **No macro hygiene issues**: no hidden name collisions or expansion pitfalls
4. **Complete coverage**: reflection enumerates ALL members; macros require manual listing (easy to miss a field)
5. **Supports consteval validation**: you can check at compile time that all member types are serializable

---

## Notes

- C++26 reflection is not yet available in production compilers as of early 2026 — EDG and experimental Clang branches have partial support.
- Libraries like **glaze** and **Boost.Describe** provide similar functionality today using C++20 techniques.
- Reflection-based serialization generates optimal code: the compiler unrolls the member loop at compile time.
