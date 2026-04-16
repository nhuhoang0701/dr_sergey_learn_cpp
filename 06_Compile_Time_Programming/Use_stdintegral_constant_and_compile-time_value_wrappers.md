# Use `std::integral_constant` and Compile-Time Value Wrappers

**Category:** Compile-Time Programming  
**Item:** #57  
**Standard:** C++11 (enhanced in C++14/C++17)  
**Reference:** <https://en.cppreference.com/w/cpp/types/integral_constant>  

---

## Topic Overview

### What Is `std::integral_constant`

`std::integral_constant` wraps a compile-time constant value into a type. This is the foundation of type-level computation in C++.

```cpp

template<class T, T v>
struct integral_constant {
    static constexpr T value = v;
    using value_type = T;
    using type = integral_constant;
    constexpr operator value_type() const noexcept { return value; }
    constexpr value_type operator()() const noexcept { return value; } // C++14
};

```

### Key Type Aliases

| Alias | Definition |
| --- | --- |
| `std::true_type` | `std::integral_constant<bool, true>` |
| `std::false_type` | `std::integral_constant<bool, false>` |
| `std::bool_constant<B>` | `std::integral_constant<bool, B>` (C++17) |

### Why Types Instead of Values

Using types to represent values enables:

| Capability | Example |
| --- | --- |
| Tag dispatch | `foo(std::true_type{})` vs `foo(std::false_type{})` |
| Trait composition | `std::is_integral<T>` inherits from `true_type` or `false_type` |
| Type-level computation | Recursive template metaprogramming with types |
| SFINAE / concepts | `enable_if_t<is_trivial_v<T>>` uses inherited `::value` |

### How Standard Traits Use It

All `<type_traits>` predicates inherit from `integral_constant`:

```cpp

// Standard library definition (simplified)
template<class T> struct is_pointer     : std::false_type {};
template<class T> struct is_pointer<T*> : std::true_type  {};

static_assert(std::is_pointer<int*>::value);   // true
static_assert(!std::is_pointer<int>::value);   // false

```

---

## Self-Assessment

### Q1: Explain how `std::true_type` and `std::false_type` are aliases for `integral_constant<bool, ...>`

```cpp

#include <iostream>
#include <type_traits>

// === true_type and false_type are just integral_constant<bool, ...> ===

// Verify at compile time
static_assert(std::is_same_v<std::true_type, std::integral_constant<bool, true>>);
static_assert(std::is_same_v<std::false_type, std::integral_constant<bool, false>>);

// They provide ::value, implicit conversion, and operator()
static_assert(std::true_type::value == true);
static_assert(std::false_type::value == false);

// === Tag Dispatch Example ===
// Different algorithms for trivially copyable vs non-trivially copyable types

template<typename T>
void copy_impl(const T* src, T* dst, std::size_t n, std::true_type /*trivially_copyable*/) {
    std::cout << "  → Using memcpy (trivially copyable)\n";
    std::memcpy(dst, src, n * sizeof(T));
}

template<typename T>
void copy_impl(const T* src, T* dst, std::size_t n, std::false_type /*not trivially_copyable*/) {
    std::cout << "  → Using element-wise copy (non-trivial)\n";
    for (std::size_t i = 0; i < n; ++i)
        dst[i] = src[i];
}

template<typename T>
void smart_copy(const T* src, T* dst, std::size_t n) {
    // Tag dispatch: the trait inherits from true_type or false_type
    copy_impl(src, dst, n, std::is_trivially_copyable<T>{});
}

// === Custom Trait Using integral_constant ===

template<typename T>
struct is_string_like : std::false_type {};

template<>
struct is_string_like<std::string> : std::true_type {};

template<>
struct is_string_like<const char*> : std::true_type {};

template<std::size_t N>
struct is_string_like<char[N]> : std::true_type {};

static_assert(is_string_like<std::string>::value);
static_assert(is_string_like<const char*>::value);
static_assert(!is_string_like<int>::value);

int main() {
    // === Implicit conversion operator ===
    std::cout << "true_type converts to: " << std::true_type{} << "\n";   // 1
    std::cout << "false_type converts to: " << std::false_type{} << "\n"; // 0

    // === operator() (C++14) ===
    constexpr auto val = std::true_type{}();
    std::cout << "true_type() returns: " << val << "\n";  // 1

    // === Tag dispatch in action ===
    int src[] = {1, 2, 3};
    int dst[3];
    std::cout << "\nCopying int[3]:\n";
    smart_copy(src, dst, 3);

    std::string s_src[] = {"hello", "world"};
    std::string s_dst[2];
    std::cout << "Copying string[2]:\n";
    smart_copy(s_src, s_dst, 2);

    // === bool_constant (C++17) ===
    using is_64bit = std::bool_constant<sizeof(void*) == 8>;
    std::cout << "\n64-bit platform: " << is_64bit::value << "\n";

    return 0;
}

```

**Expected output (64-bit, trivially-copyable string impl may vary):**

```text

true_type converts to: 1
false_type converts to: 0
true_type() returns: 1

Copying int[3]:
  → Using memcpy (trivially copyable)
Copying string[2]:
  → Using element-wise copy (non-trivial)

64-bit platform: 1

```

### Q2: Write a type list with variadic templates and extract the Nth type

```cpp

#include <iostream>
#include <type_traits>
#include <cstddef>

// === Type List ===
template<typename... Ts>
struct type_list {
    static constexpr std::size_t size = sizeof...(Ts);
};

// === Extract Nth type (recursive) ===
template<std::size_t N, typename List>
struct type_at;

template<std::size_t N, typename Head, typename... Tail>
struct type_at<N, type_list<Head, Tail...>>
    : type_at<N - 1, type_list<Tail...>> {};

template<typename Head, typename... Tail>
struct type_at<0, type_list<Head, Tail...>> {
    using type = Head;
};

template<std::size_t N, typename List>
using type_at_t = typename type_at<N, List>::type;

// === Contains check ===
template<typename T, typename List>
struct contains;

template<typename T>
struct contains<T, type_list<>> : std::false_type {};

template<typename T, typename Head, typename... Tail>
struct contains<T, type_list<Head, Tail...>>
    : std::bool_constant<std::is_same_v<T, Head> || contains<T, type_list<Tail...>>::value> {};

// === Index of type ===
template<typename T, typename List>
struct index_of;

template<typename T, typename Head, typename... Tail>
struct index_of<T, type_list<Head, Tail...>>
    : std::integral_constant<std::size_t,
          std::is_same_v<T, Head> ? 0 : 1 + index_of<T, type_list<Tail...>>::value> {};

// Compile-time tests
using my_types = type_list<int, double, char, std::string>;

static_assert(my_types::size == 4);
static_assert(std::is_same_v<type_at_t<0, my_types>, int>);
static_assert(std::is_same_v<type_at_t<1, my_types>, double>);
static_assert(std::is_same_v<type_at_t<2, my_types>, char>);
static_assert(std::is_same_v<type_at_t<3, my_types>, std::string>);

static_assert(contains<int, my_types>::value);
static_assert(contains<double, my_types>::value);
static_assert(!contains<float, my_types>::value);

static_assert(index_of<int, my_types>::value == 0);
static_assert(index_of<char, my_types>::value == 2);

int main() {
    std::cout << "=== Type List Operations ===\n";
    std::cout << "List size: " << my_types::size << "\n";

    std::cout << "\ntype_at<0> = " << typeid(type_at_t<0, my_types>).name() << "\n";
    std::cout << "type_at<1> = " << typeid(type_at_t<1, my_types>).name() << "\n";
    std::cout << "type_at<2> = " << typeid(type_at_t<2, my_types>).name() << "\n";
    std::cout << "type_at<3> = " << typeid(type_at_t<3, my_types>).name() << "\n";

    std::cout << "\ncontains<int>:    " << contains<int, my_types>::value << "\n";
    std::cout << "contains<float>:  " << contains<float, my_types>::value << "\n";

    std::cout << "\nindex_of<int>:    " << index_of<int, my_types>::value << "\n";
    std::cout << "index_of<char>:   " << index_of<char, my_types>::value << "\n";

    // === Using integral_constant for recursive computation ===
    // Count types matching a predicate
    std::cout << "\n=== How integral_constant enables recursion ===\n";
    std::cout << "Each recursive step returns a TYPE (integral_constant<size_t, N>)\n";
    std::cout << "The compiler resolves the chain at compile time.\n";

    return 0;
}

```

**Expected output (type names are compiler-dependent):**

```text

=== Type List Operations ===
List size: 4

type_at<0> = i
type_at<1> = d
type_at<2> = c
type_at<3> = NSt7__cxx1112basic_stringIcSt11char_traitsIcESaIcEEE

contains<int>:    1
contains<float>:  0

index_of<int>:    0
index_of<char>:   2

=== How integral_constant enables recursion ===
Each recursive step returns a TYPE (integral_constant<size_t, N>)
The compiler resolves the chain at compile time.

```

### Q3: Build a compile-time map from types to integer values using template specializations

```cpp

#include <iostream>
#include <type_traits>
#include <string>

// === Compile-Time Type → Value Map ===
// Uses integral_constant to associate each type with a unique integer ID

// Primary template: no mapping (will cause compile error if used for unmapped type)
template<typename T>
struct type_id; // intentionally undefined — SFINAE-friendly

// Specializations: map types to integer IDs
template<> struct type_id<int>         : std::integral_constant<int, 1> {};
template<> struct type_id<double>      : std::integral_constant<int, 2> {};
template<> struct type_id<char>        : std::integral_constant<int, 3> {};
template<> struct type_id<std::string> : std::integral_constant<int, 4> {};
template<> struct type_id<float>       : std::integral_constant<int, 5> {};
template<> struct type_id<bool>        : std::integral_constant<int, 6> {};

// Convenient alias
template<typename T>
inline constexpr int type_id_v = type_id<T>::value;

// Compile-time verification
static_assert(type_id_v<int> == 1);
static_assert(type_id_v<double> == 2);
static_assert(type_id_v<std::string> == 4);

// === Type → String Name Map (using pointer constant) ===
template<typename T>
struct type_name;

template<> struct type_name<int>    { static constexpr const char* value = "int"; };
template<> struct type_name<double> { static constexpr const char* value = "double"; };
template<> struct type_name<char>   { static constexpr const char* value = "char"; };
template<> struct type_name<float>  { static constexpr const char* value = "float"; };
template<> struct type_name<bool>   { static constexpr const char* value = "bool"; };

// === SFINAE check: does a type have a mapping? ===
template<typename T, typename = void>
struct has_type_id : std::false_type {};

template<typename T>
struct has_type_id<T, std::void_t<decltype(type_id<T>::value)>> : std::true_type {};

static_assert(has_type_id<int>::value);
static_assert(has_type_id<double>::value);
static_assert(!has_type_id<long long>::value);  // not mapped

// === Practical: Serialization Tag ===
template<typename T>
void serialize(const T& val) {
    static_assert(has_type_id<T>::value, "Type must be registered in type_id map");
    // Write type tag then value
    std::cout << "[tag=" << type_id_v<T> << "] ";
    if constexpr (std::is_same_v<T, std::string>)
        std::cout << "\"" << val << "\"\n";
    else
        std::cout << val << "\n";
}

int main() {
    std::cout << "=== Compile-Time Type → ID Map ===\n";
    std::cout << "int    → " << type_id_v<int> << "\n";
    std::cout << "double → " << type_id_v<double> << "\n";
    std::cout << "char   → " << type_id_v<char> << "\n";
    std::cout << "string → " << type_id_v<std::string> << "\n";
    std::cout << "float  → " << type_id_v<float> << "\n";
    std::cout << "bool   → " << type_id_v<bool> << "\n";

    std::cout << "\n=== SFINAE: has_type_id ===\n";
    std::cout << "int:       " << has_type_id<int>::value << "\n";
    std::cout << "long long: " << has_type_id<long long>::value << "\n";

    std::cout << "\n=== Practical: Tagged Serialization ===\n";
    serialize(42);
    serialize(3.14);
    serialize(std::string("hello"));

    // === integral_constant implicit conversion ===
    std::cout << "\n=== Implicit Conversion ===\n";
    // type_id<int>{} converts to int via operator value_type()
    int id = type_id<int>{};
    std::cout << "type_id<int>{} as int: " << id << "\n";

    // Can use in switch
    switch (type_id_v<double>) {
        case 1: std::cout << "int\n"; break;
        case 2: std::cout << "double\n"; break;
        default: std::cout << "other\n"; break;
    }

    return 0;
}

```

**Expected output:**

```text

=== Compile-Time Type → ID Map ===
int    → 1
double → 2
char   → 3
string → 4
float  → 5
bool   → 6

=== SFINAE: has_type_id ===
int:       1
long long: 0

=== Practical: Tagged Serialization ===
[tag=1] 42
[tag=2] 3.14
[tag=4] "hello"

=== Implicit Conversion ===
type_id<int>{} as int: 1
double

```

---

## Notes

- `std::integral_constant<T, v>` turns a compile-time value `v` into a type — the basis of all type traits.
- `std::true_type` / `std::false_type` are the boolean specializations used by every `<type_traits>` predicate.
- Tag dispatch uses `true_type{}` / `false_type{}` as function arguments to select overloads without `if constexpr`.
- `std::bool_constant<B>` (C++17) simplifies `integral_constant<bool, B>`.
- The implicit `operator value_type()` lets you use `integral_constant` instances directly in boolean/numeric contexts.
- Custom type-to-value maps via specialization give zero-overhead compile-time lookup tables.
