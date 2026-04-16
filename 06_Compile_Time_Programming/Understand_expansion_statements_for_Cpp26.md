# Understand expansion statements for C++26

**Category:** Compile-Time Programming  
**Standard:** C++26  
**Reference:** <https://wg21.link/P1306>  

---

## Topic Overview

Expansion statements (`for...` over packs and tuples) allow iterating over parameter packs and compile-time sequences directly in regular code, replacing recursive template metaprogramming.

### Current Workarounds (Pre-C++26)

```cpp

#include <tuple>
#include <iostream>

// To iterate a tuple, you need:
// 1. index_sequence trick
template<typename Tuple, size_t... Is>
void print_impl(const Tuple& t, std::index_sequence<Is...>) {
    ((std::cout << std::get<Is>(t) << " "), ...);
}

template<typename... Ts>
void print_tuple(const std::tuple<Ts...>& t) {
    print_impl(t, std::index_sequence_for<Ts...>{});
}

```

### C++26 Expansion Statements

```cpp

#include <tuple>
#include <iostream>

// C++26: direct iteration over tuples and packs
template<typename... Ts>
void print_all(const std::tuple<Ts...>& t) {
    template for (const auto& elem : t) {
        std::cout << elem << " ";
    }
}

// Also works with parameter packs:
template<typename... Args>
void log(Args&&... args) {
    template for (auto&& arg : {args...}) {
        std::cout << arg << "\n";
    }
}

```

### Compile-Time Conditional in Expansion

```cpp

#include <tuple>
#include <type_traits>
#include <iostream>

template<typename... Ts>
void print_ints_only(const std::tuple<Ts...>& t) {
    template for (const auto& elem : t) {
        if constexpr (std::is_integral_v<std::remove_cvref_t<decltype(elem)>>) {
            std::cout << elem << " ";
        }
    }
}

auto t = std::tuple{42, "hello", 3.14, 99};
print_ints_only(t);  // Prints: 42 99

```

---

## Self-Assessment

### Q1: How does `template for` differ from a regular for loop

`template for` unrolls at compile time — each iteration may have a different type. A regular for loop requires a single type for the loop variable. `template for` is essentially a compile-time fold over heterogeneous collections.

### Q2: What can you iterate over with expansion statements

Tuples, parameter packs, and `std::integer_sequence`. Anything that has a compile-time-known size and heterogeneous element types.

### Q3: What does this replace

Index sequence tricks, fold expressions (for complex operations), recursive template instantiation, `std::apply`, and `std::tuple_cat` workarounds. Expansion statements make tuple/pack iteration as simple as iterating a range.

---

## Notes

- `template for` is the keyword combination (subject to change before C++26 ships).
- Each "iteration" instantiates the loop body with a different type.
- Massively simplifies reflection-based code generation (combined with P2996).
- Reduces compile times vs recursive template instantiation.
