# Enumerate struct members at compile time using reflection

**Category:** Reflection (C++26)  
**Item:** #534  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++26 reflection lets you iterate over all data members of a struct at compile time using `std::meta::nonstatic_data_members_of(^S)`. This enables writing generic transformations without macros.

| API | Purpose |
| --- | --- |
| `^MyStruct` | Reflect the struct type |
| `nonstatic_data_members_of(^S)` | Get all non-static data members |
| `members_of(^S)` | Get ALL members (including static, functions) |
| `identifier_of(m)` | Get member name |
| `type_of(m)` | Get member type as `meta::info` |
| `obj.[:m:]` | Access member on an object via splice |

---

## Self-Assessment

### Q1: Iterate non-static data members with `members_of`

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>
#include <string_view>

struct Point {
    double x;
    double y;
    double z;
};

int main() {
    constexpr auto members = std::meta::nonstatic_data_members_of(^Point);

    std::cout << "Point has " << members.size() << " members:\n";
    template for (constexpr auto m : members) {
        std::cout << "  " << std::meta::identifier_of(m)
                  << " : " << std::meta::display_string_of(std::meta::type_of(m))
                  << '\n';
    }

    // Access members on an object:
    Point p{1.0, 2.0, 3.0};
    template for (constexpr auto m : members) {
        std::cout << std::meta::identifier_of(m)
                  << " = " << p.[:m:] << '\n';
    }
}
// Output:
// Point has 3 members:
//   x : double
//   y : double
//   z : double
// x = 1
// y = 2
// z = 3

```

### Q2: Generic `to_tuple` using reflection

```cpp

#include <meta>
#include <tuple>
#include <iostream>
#include <string>

struct Person {
    std::string name;
    int age;
    double height;
};

// Generic to_tuple for ANY aggregate:
template <typename T>
auto to_tuple(const T& obj) {
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);

    // Build the tuple by splicing each member access:
    return [&]<std::size_t... Is>(std::index_sequence<Is...>) {
        return std::make_tuple(obj.[:members[Is]:]...);
    }(std::make_index_sequence<members.size()>{});
}

int main() {
    Person p{"Alice", 30, 5.6};
    auto t = to_tuple(p);

    std::cout << std::get<0>(t) << ", "
              << std::get<1>(t) << ", "
              << std::get<2>(t) << '\n';
    // Output: Alice, 30, 5.6

    // Works for ANY struct:
    struct Vec3 { float x, y, z; };
    Vec3 v{1.0f, 2.0f, 3.0f};
    auto vt = to_tuple(v);
    // vt is tuple<float, float, float>{1.0f, 2.0f, 3.0f}
}

```

### Q3: Compile-time field counter without macros

```cpp

#include <meta>
#include <iostream>

struct Empty {};
struct One { int a; };
struct Three { int a; double b; char c; };
struct Complex {
    std::string name;
    int value;
    std::vector<int> data;
    double factor;
};

// Generic field counter:
template <typename T>
constexpr std::size_t field_count() {
    return std::meta::nonstatic_data_members_of(^T).size();
}

static_assert(field_count<Empty>() == 0);
static_assert(field_count<One>() == 1);
static_assert(field_count<Three>() == 3);
static_assert(field_count<Complex>() == 4);

// Practical use: compile-time validation
template <typename T>
    requires (field_count<T>() <= 8)
struct SmallStruct {
    T data;
};

// SmallStruct<Complex> ok;       // 4 fields, OK
// SmallStruct<HugeStruct> fail;  // >8 fields, compile error

int main() {
    std::cout << "Empty: " << field_count<Empty>() << '\n';    // 0
    std::cout << "Three: " << field_count<Three>() << '\n';    // 3
    std::cout << "Complex: " << field_count<Complex>() << '\n'; // 4
}

```

---

## Notes

- `nonstatic_data_members_of` returns only instance fields (no static members).
- `members_of` returns everything including static members, member functions, etc.
- `template for` is the expansion statement for iterating compile-time sequences.
- Reflection replaces C++17 structured bindings hacks for field counting.
- All reflection queries are `consteval` — zero runtime overhead.
