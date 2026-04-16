# Generate code from reflections using template splicing [:...]:

**Category:** Reflection (C++26)  
**Item:** #617  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/reflection>  

---

## Topic Overview

The **splice operator** `[:r:]` converts a compile-time reflection (`std::meta::info`) back into code. It bridges the gap between the meta world (reflections) and the code world (types, values, expressions).

| Splice form | Produces |
| --- | --- |
| `[:type_reflection:]` | A type |
| `[:value_reflection:]` | A value/expression |
| `obj.[:member_reflection:]` | Member access |
| `[:func_reflection:](args)` | Function call |

```cpp

Reflect → Transform → Splice:

  int x = 42;
  ^x  ─────→ meta::info  ───→ [:^x:] == 42
  ^int ────→ meta::info  ───→ using T = [:^int:]; // T is int

```

---

## Self-Assessment

### Q1: Splice a reflected type into a template argument

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <vector>
#include <string>

template <typename T>
void print_type() {
    std::cout << std::meta::display_string_of(^T) << '\n';
}

int main() {
    // Reflect a type:
    constexpr auto int_refl = ^int;
    constexpr auto str_refl = ^std::string;

    // Splice back into code:
    using T1 = [:int_refl:];   // T1 is int
    using T2 = [:str_refl:];   // T2 is std::string

    T1 x = 42;
    T2 s = "hello";
    std::cout << x << ", " << s << '\n';  // 42, hello

    // Use in template arguments:
    std::vector<[:int_refl:]> v = {1, 2, 3};  // vector<int>

    // Build types from reflections dynamically:
    constexpr auto members = std::meta::nonstatic_data_members_of(^struct { int a; double b; });
    using FirstType = [:std::meta::type_of(members[0]):];
    static_assert(std::is_same_v<FirstType, int>);
}

```

### Q2: Call a function by its reflected name

```cpp

#include <meta>
#include <iostream>
#include <string_view>

int add(int a, int b) { return a + b; }
int mul(int a, int b) { return a * b; }
int sub(int a, int b) { return a - b; }

// Dispatch table built from reflections:
struct MathOps {
    static int add(int a, int b) { return a + b; }
    static int mul(int a, int b) { return a * b; }
    static int sub(int a, int b) { return a - b; }
};

template <typename OpsType>
constexpr int dispatch(std::string_view name, int a, int b) {
    constexpr auto members = std::meta::members_of(^OpsType);
    template for (constexpr auto m : members) {
        if (std::meta::identifier_of(m) == name) {
            return [:m:](a, b);  // splice + call!
        }
    }
    return -1;  // not found
}

int main() {
    // Compile-time lookup, runtime dispatch:
    std::cout << dispatch<MathOps>("add", 10, 20) << '\n';  // 30
    std::cout << dispatch<MathOps>("mul", 10, 20) << '\n';  // 200
    std::cout << dispatch<MathOps>("sub", 10, 20) << '\n';  // -10
}

```

### Q3: Type-safe serializer using member iteration + type splicing

```cpp

#include <meta>
#include <iostream>
#include <string>
#include <sstream>
#include <type_traits>

// Serializer that handles types differently using spliced type info:
template <typename T>
std::string serialize(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::ostringstream oss;
    oss << "{";
    bool first = true;

    template for (constexpr auto m : members) {
        if (!first) oss << ", ";
        first = false;

        // Splice the member type:
        using MType = [:std::meta::type_of(m):];

        oss << '"' << std::meta::identifier_of(m) << "": ";

        if constexpr (std::is_arithmetic_v<MType>) {
            oss << obj.[:m:];
        } else if constexpr (std::is_same_v<MType, std::string>) {
            oss << '"' << obj.[:m:] << '"';
        } else if constexpr (std::is_same_v<MType, bool>) {
            oss << (obj.[:m:] ? "true" : "false");
        } else {
            // Recursively serialize nested structs:
            oss << serialize(obj.[:m:]);
        }
    }
    oss << "}";
    return oss.str();
}

struct Address {
    std::string city;
    int zip;
};

struct Person {
    std::string name;
    int age;
    Address address;
};

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << serialize(p) << '\n';
    // Output: {"name": "Alice", "age": 30, "address": {"city": "NYC", "zip": 10001}}
}

```

---

## Notes

- `[:r:]` requires `r` to be a `consteval`-computed `std::meta::info`.
- Splicing types: `[:type_info:]` produces a type usable with `using`, template args, etc.
- Splicing values: `[:value_info:]` produces an expression.
- Splicing members: `obj.[:member_info:]` accesses the reflected member on `obj`.
- Splicing functions: `[:func_info:](args...)` calls the reflected function.
- This replaces code generation tools and extensive template metaprogramming.
