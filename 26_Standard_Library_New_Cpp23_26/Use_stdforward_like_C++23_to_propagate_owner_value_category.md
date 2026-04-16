# Use `std::forward_like` (C++23) to Propagate Owner Value Category

**Category:** Standard Library — New in C++23/26  
**Standard:** C++23  
**Reference:** [cppreference — std::forward_like](https://en.cppreference.com/w/cpp/utility/forward_like)  

---

## Topic Overview

`std::forward_like<Owner>(member)` applies the value category and const-qualification of `Owner` to `member`. It solves a long-standing problem in generic code: when you have a forwarding reference to an object and want to access one of its members, `std::forward` alone cannot correctly propagate the owner's rvalue-ness to the member.

Consider a generic `get()` function. If the owner is an rvalue, the member should also be treated as an rvalue (enabling moves). If the owner is an lvalue, the member should remain an lvalue. Before C++23, achieving this required complex `decltype` gymnastics or manual overload sets.

| Owner type `T&&` | Member type `M` | `std::forward_like<T>(m)` result |
| --- | --- | --- |
| `Owner&` (lvalue) | `M` | `M&` |
| `Owner&&` (rvalue) | `M` | `M&&` |
| `const Owner&` | `M` | `const M&` |
| `const Owner&&` | `M` | `const M&&` |

The mental model: **"forward the member *as if* it were the owner."** The owner's reference and const qualifiers are transferred to the member, overriding the member's own qualifiers.

```cpp

                    Owner category
                   ┌──────────────┐
                   │  T&   T&&    │
                   │  ↓     ↓     │
   forward_like<T>(member)        │
                   │  ↓     ↓     │
                   │  M&   M&&    │  ← result category
                   └──────────────┘

```

This is essential for implementing tuple-like types, `std::optional`-like wrappers, any type that stores a value and provides accessors in generic contexts.

---

## Self-Assessment

### Q1: What problem does `std::forward_like` solve that `std::forward` cannot

```cpp

#include <utility>
#include <string>
#include <iostream>
#include <type_traits>

template <typename T>
struct Wrapper {
    T value;

    // ─── PRE-C++23: manual overloads needed ───
    // T&        get() &       { return value; }
    // const T&  get() const&  { return value; }
    // T&&       get() &&      { return std::move(value); }
    // const T&& get() const&& { return std::move(value); }

    // ─── C++23: single deducing-this function ───
    template <typename Self>
    auto&& get(this Self&& self) {
        return std::forward_like<Self>(self.value);
    }
};

int main() {
    Wrapper<std::string> w{"hello"};
    const Wrapper<std::string> cw{"world"};

    // lvalue → lvalue ref
    auto& ref = w.get();
    static_assert(std::is_lvalue_reference_v<decltype(w.get())>);

    // const lvalue → const lvalue ref
    static_assert(std::is_const_v<
        std::remove_reference_t<decltype(cw.get())>>);

    // rvalue → rvalue ref (enables move)
    std::string stolen = std::move(w).get();
    std::cout << "Moved: " << stolen << '\n';

    // const rvalue → const rvalue ref (no move possible)
    static_assert(std::is_rvalue_reference_v<
        decltype(std::move(cw).get())>);
}

```

**Key insight:** `std::forward<Self>(self).value` would give `T&` or `T&&` depending on `Self`, but if `T` itself is a reference type, the behavior breaks. `std::forward_like<Self>(self.value)` handles all combinations correctly.

---

### Q2: How do you use `std::forward_like` in a generic `Optional`-like type

```cpp

#include <utility>
#include <optional>
#include <string>
#include <iostream>
#include <stdexcept>

template <typename T>
class Optional {
    alignas(T) unsigned char storage_[sizeof(T)];
    bool engaged_ = false;

    T&       get_ref()       { return *reinterpret_cast<T*>(storage_); }
    const T& get_ref() const { return *reinterpret_cast<const T*>(storage_); }

public:
    Optional() = default;

    template <typename U>
    Optional(U&& val) : engaged_(true) {
        ::new (storage_) T(std::forward<U>(val));
    }

    ~Optional() { if (engaged_) get_ref().~T(); }

    explicit operator bool() const { return engaged_; }

    // Deducing this + forward_like: single accessor for all 4 overloads
    template <typename Self>
    auto&& value(this Self&& self) {
        if (!self.engaged_)
            throw std::runtime_error("bad optional access");
        return std::forward_like<Self>(self.get_ref());
    }

    // Map/transform with correct forwarding
    template <typename Self, typename F>
    auto transform(this Self&& self, F&& f) {
        using U = std::invoke_result_t<F,
            decltype(std::forward_like<Self>(self.get_ref()))>;
        if (!self.engaged_) return Optional<U>{};
        return Optional<U>{
            std::forward<F>(f)(
                std::forward_like<Self>(self.get_ref()))};
    }
};

int main() {
    Optional<std::string> opt{"hello"};

    // lvalue access — no move
    std::cout << opt.value() << '\n';

    // rvalue access — moves the string out
    std::string taken = std::move(opt).value();
    std::cout << "Taken: " << taken << '\n';

    // transform chains
    Optional<std::string> name{"Alice"};
    auto length = name.transform([](const std::string& s) {
        return s.size();
    });
    std::cout << "Length: " << length.value() << '\n';
}

```

---

### Q3: What is the exact specification of `std::forward_like`, and how does it differ from naive alternatives

```cpp

#include <utility>
#include <type_traits>
#include <tuple>
#include <string>
#include <iostream>

// ═══ Simplified implementation of forward_like ═══
// The standard says: copy the ref+const qualifiers of Owner onto Member
namespace detail {
    template <typename Owner, typename Member>
    constexpr auto&& forward_like_impl(Member&& m) noexcept {
        // 1. Determine if Owner is an rvalue reference
        constexpr bool is_rvalue = !std::is_lvalue_reference_v<Owner>;
        // 2. Determine if Owner is const
        using OwnerUnref = std::remove_reference_t<Owner>;
        constexpr bool is_const = std::is_const_v<OwnerUnref>;

        using M = std::remove_reference_t<Member>;
        if constexpr (is_const && is_rvalue)
            return static_cast<const M&&>(m);
        else if constexpr (is_const)
            return static_cast<const M&>(m);
        else if constexpr (is_rvalue)
            return static_cast<M&&>(m);
        else
            return static_cast<M&>(m);
    }
}

// ═══ Practical use: generic tuple-like get ═══
template <std::size_t I, typename Tuple>
auto&& get_element(Tuple&& t) {
    // forward_like propagates the value category of the tuple
    // to the individual element
    return std::forward_like<Tuple>(std::get<I>(t));
}

int main() {
    auto tup = std::make_tuple(std::string("hello"), 42);

    // lvalue tuple → lvalue element
    auto& s1 = get_element<0>(tup);
    s1 += " world";  // modifies in place

    // rvalue tuple → rvalue element (movable)
    auto s2 = get_element<0>(std::move(tup));
    std::cout << s2 << '\n';  // "hello world"

    // Verify const propagation
    const auto ctup = std::make_tuple(std::string("const"), 0);
    static_assert(std::is_const_v<
        std::remove_reference_t<decltype(get_element<0>(ctup))>>);
}

```

---

## Notes

- `std::forward_like` is in `<utility>`. Signature: `template<class T, class U> constexpr auto&& forward_like(U&& x) noexcept;`
- It pairs naturally with **deducing this** (C++23 explicit object parameters) to replace 4-overload accessor sets with a single function.
- **Mental model:** `forward_like<T>(x)` means "give `x` the same ref-qualification and const-ness as `T`."
- Unlike `std::forward`, it does *not* require `x` to be a forwarding reference — it works on any expression.
- The const-rvalue case (`const T&&`) is preserved, which matters for types that distinguish it (rare but valid).
- Use it whenever you write generic code that accesses a *member* through a forwarding reference to the *owner*.
- Feature-test macro: `__cpp_lib_forward_like >= 202207L`.
- Compilers: GCC 12+, Clang 16+, MSVC 19.34+ (VS 2022 17.4).
