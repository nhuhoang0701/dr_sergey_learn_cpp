# Use std::is_within_lifetime (C++26) for constexpr lifetime queries

**Category:** Standard Library Utilities  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2641>  

---

## Topic Overview

`std::is_within_lifetime` is a `consteval` function that checks whether a pointer refers to an object whose lifetime has begun and not yet ended. This enables compile-time inspection of union members and optional-like types.

The reason this matters is subtle: unions have been tricky to use at compile time because the compiler knows which member is "active" but there was previously no way to query that from constexpr code. C++26 plugs that gap.

### The Problem It Solves

Before C++26, if you had a union inside a `consteval` function, you had no standard way to ask "which member is currently live?" You had to track it manually with a boolean flag or discriminant. With `std::is_within_lifetime`, you can ask the compiler directly.

```cpp
#include <type_traits>

union IntOrFloat {
    int i;
    float f;
};

// At compile time, which union member is active?
// Before C++26: no way to query this in constexpr context
// After C++26: std::is_within_lifetime tells you

consteval int get_active_int(const IntOrFloat& u) {
    if (std::is_within_lifetime(&u.i))
        return u.i;
    // u.i is not the active member
    return -1;
}
```

Notice that this function is `consteval` - it only exists at compile time. At runtime, reading an inactive union member is undefined behavior, so the compiler correctly refuses to let you call this outside of constant evaluation.

### Practical Use: constexpr Optional

Here is where `is_within_lifetime` earns its keep. Implementing a fully `constexpr` optional-like type requires the ability to tell whether `value_` was ever constructed. Without `is_within_lifetime`, you fall back to a manual `has_value_` flag. With it, the compiler's own lifetime tracking can serve as the source of truth.

```cpp
#include <memory>

template<typename T>
class ConstexprOptional {
    union { T value_; };
    bool has_value_ = false;

public:
    constexpr ConstexprOptional() = default;
    constexpr ConstexprOptional(T val) : value_(std::move(val)), has_value_(true) {}

    constexpr bool has_value() const {
        // C++26: can verify at compile time:
        // return std::is_within_lifetime(&value_);
        return has_value_;
    }

    constexpr const T& value() const { return value_; }

    constexpr ~ConstexprOptional() {
        if (has_value_) value_.~T();
    }
};
```

The comment shows where `is_within_lifetime` would replace the manual flag - a cleaner design that eliminates the risk of the flag and the union getting out of sync.

---

## Self-Assessment

### Q1: Why is this only useful at compile time

At runtime, accessing an inactive union member is UB - the program is already broken before you can check. At compile time, the compiler tracks which objects are alive, so it can answer the query precisely. This enables constexpr code that operates on unions.

The key insight is that the compiler maintains a separate model of object lifetimes during constant evaluation. `is_within_lifetime` is essentially a hook into that internal model. At runtime there is no equivalent bookkeeping, so the function simply does not exist there.

### Q2: What are the use cases

- `constexpr std::optional` implementation
- `constexpr std::variant` active member detection
- Compile-time serialization/deserialization with tagged unions
- Type-punning safety checks in consteval functions

### Q3: Can this be used at runtime

No. `std::is_within_lifetime` is `consteval` - it can only be called during constant evaluation. At runtime, you must track active members manually (e.g., with a discriminant/tag).

---

## Notes

- `is_within_lifetime` is a compiler magic function - only the compiler knows object lifetimes.
- Enables correct constexpr implementations of `optional`, `variant`, and `any`.
- Part of the broader C++26 effort to make more standard library types fully constexpr.
- Complements `std::start_lifetime_as` (C++23) which creates objects.
