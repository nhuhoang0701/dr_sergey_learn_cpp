# Provide customization points via ADL, tag_invoke, or CPO patterns

**Category:** API & Library Design  
**Standard:** C++20/23  
**Reference:** <https://wg21.link/P1895R0>  

---

## Topic Overview

A **customization point** is a function that library code calls but users can override for their types. Three patterns exist, from simplest to most robust.

### Pattern 1: ADL (the classic "swap" pattern)

```cpp

#include <utility>
#include <iostream>

namespace lib {
    // Default implementation
    template<typename T>
    void serialize(const T& val, std::ostream& os) {
        os << val;  // Default: use operator<<
    }
}

namespace user {
    struct Point { double x, y; };

    // Customization: found via ADL
    void serialize(const Point& p, std::ostream& os) {
        os << "(" << p.x << "," << p.y << ")";
    }
}

template<typename T>
void log_value(const T& val) {
    using lib::serialize;  // Bring default into scope
    serialize(val, std::cout);  // ADL finds user::serialize for user types
    std::cout << "\n";
}

```

### Pattern 2: tag_invoke (proposed for std::execution)

```cpp

#include <iostream>
#include <type_traits>

// A tag type identifies the operation
struct serialize_fn {
    template<typename T>
    void operator()(const T& val, std::ostream& os) const {
        // Dispatch via tag_invoke
        tag_invoke(*this, val, os);
    }
};
inline constexpr serialize_fn serialize{};

// Default via tag_invoke
template<typename T>
void tag_invoke(serialize_fn, const T& val, std::ostream& os) {
    os << val;
}

// User customization
namespace user {
    struct Widget { int id; };
    void tag_invoke(serialize_fn, const Widget& w, std::ostream& os) {
        os << "Widget#" << w.id;
    }
}

```

### Pattern 3: CPO (Customization Point Object) — Used by Ranges

```cpp

#include <concepts>
#include <iostream>

namespace lib {
    namespace _detail {
        // Fallback
        void serialize(auto const&, auto&) = delete;

        struct _serialize_fn {
            template<typename T, typename OS>
            requires requires(const T& t, OS& os) {
                serialize(t, os);  // ADL-found serialize
            }
            void operator()(const T& val, OS& os) const {
                serialize(val, os);  // Call ADL-found version
            }
        };
    }
    inline constexpr _detail::_serialize_fn serialize{};
}

// User just provides an ADL-visible serialize and the CPO finds it

```

---

## Self-Assessment

### Q1: Compare the three customization point patterns

| Pattern | Pros | Cons |
| --- | --- | --- |
| ADL | Simple, well-understood | Hijacking risk, `using` dance |
| tag_invoke | No hijacking, single dispatch point | Verbose, non-standard |
| CPO | Type-safe, no hijacking | Complex to implement |

### Q2: What is "ADL hijacking" and how do CPOs prevent it

ADL hijacking occurs when an unrelated namespace accidentally provides a function matching the customization point name. CPOs prevent this by wrapping the dispatch in a function object — the user calls `lib::serialize(x, os)` (qualified), and the CPO internally controls how ADL is used.

### Q3: Show why `using std::swap; swap(a, b);` is necessary

Without the `using` declaration, unqualified `swap(a, b)` only finds ADL-visible overloads. If `a` and `b` are built-in types, ADL finds nothing. The `using std::swap` brings the fallback into scope, and ADL can still find a better match for user types.

---

## Notes

- `std::ranges` uses CPOs everywhere: `ranges::begin`, `ranges::end`, `ranges::swap`.
- `std::execution` (P2300) uses `tag_invoke` for sender/receiver customization.
- For new libraries, CPOs are the recommended approach (no hijacking, composable).
- ADL is still fine for simple customization points like `swap`, `operator<<`.
