# Know argument-dependent lookup (ADL) and its implications

**Category:** Core Language Fundamentals  
**Standard:** C++98 (fundamental rule, refined through C++20/23)  
**Reference:** <https://en.cppreference.com/w/cpp/language/adl>  

---

## Topic Overview

### What Is ADL

**Argument-Dependent Lookup (ADL)**, also known as **Koenig Lookup** (named after Andrew Koenig), is a set of rules that tell the compiler where to look for **unqualified function names** based on the **types of the arguments** passed to the function.

When you call a function without qualifying it with a namespace (e.g., `func(x)` instead of `ns::func(x)`), the compiler searches:

1. **Normal unqualified lookup** — the enclosing scopes, `using` declarations, etc.
2. **ADL** — the **associated namespaces** of each argument type.

The results from both lookups are merged into one overload set, and overload resolution picks the best match.

### Why Does ADL Exist

Without ADL, you would need to write:

```cpp

std::operator<<(std::cout, "hello");  // Ugly!

```

ADL allows the natural syntax:

```cpp

std::cout << "hello";  // operator<< found via ADL from std::cout's namespace

```

### Associated Namespaces Rules

For a given type `T`, ADL searches these namespaces:

| Type of argument | Associated namespaces |
| --- | --- |
| Fundamental type (`int`, `double`) | None (no ADL) |
| Class type `ns::C` | `ns`, plus namespaces of base classes |
| Class template `ns::C<ns2::T>` | `ns` and `ns2` |
| Enumeration `ns::E` | `ns` |
| Pointer to `T` / reference to `T` | Same as `T` |
| Function type `R(A1,A2)` | Union of associated namespaces of `R`, `A1`, `A2` |

### Key Rules and Edge Cases

1. **ADL only applies to unqualified calls** — `ns::f(x)` never triggers ADL.
2. **ADL does NOT apply if an object, variable, or type with the same name is found** in the local scope.
3. **ADL can find `friend` functions** that are only declared inside a class definition (the "hidden friend" idiom).
4. **ADL ignores `using` directives and declarations** when building the candidate set — it directly searches the associated namespaces.
5. **Template functions** need a visible declaration for ADL to find them (two-phase lookup interaction).

### The Hidden Friend Idiom

A function defined as `friend` inside a class body, with no out-of-class declaration, can **only** be found by ADL:

```cpp

namespace geom {
    struct Point {
        double x, y;
        // Hidden friend — only callable via ADL
        friend bool operator==(const Point& a, const Point& b) {
            return a.x == b.x && a.y == b.y;
        }
        friend std::ostream& operator<<(std::ostream& os, const Point& p) {
            return os << "(" << p.x << ", " << p.y << ")";
        }
    };
}

int main() {
    geom::Point a{1, 2}, b{3, 4};
    std::cout << a << "\n";   // Found via ADL (argument is geom::Point)
    bool eq = (a == b);        // Found via ADL
    // geom::operator==(a, b); // ERROR — hidden friend has no namespace-scope declaration
}

```

**Benefits of hidden friends:**

- Reduces the overload set (fewer candidates → faster compilation)
- Prevents implicit conversions on the left-hand side
- Cleaner encapsulation

### ADL and `swap` — The Classic Pattern

```cpp

namespace lib {
    struct Widget {
        int* data;
        std::size_t size;

        friend void swap(Widget& a, Widget& b) noexcept {
            using std::swap;
            std::swap(a.data, b.data);
            std::swap(a.size, b.size);
        }
    };
}

template<typename T>
void generic_sort_helper(T& a, T& b) {
    using std::swap;  // Bring std::swap into scope as fallback
    if (b < a)
        swap(a, b);   // ADL finds lib::swap for lib::Widget, std::swap for others
}

```

The `using std::swap; swap(a, b);` two-step pattern is essential in generic code.

### ADL Pitfall: Unexpected Overload Selection

```cpp

namespace evil {
    struct Token {};
    void process(Token, int) { std::cout << "evil::process\n"; }
}

namespace app {
    void process(evil::Token, double) { std::cout << "app::process\n"; }

    void run() {
        evil::Token t;
        process(t, 42);  // Finds BOTH evil::process(Token, int) via ADL
                          // AND app::process(Token, double) via normal lookup
                          // evil::process wins (int is exact match for 42)
    }
}

```

### Suppressing ADL

Use **qualified calls** or **(parentheses around the function name)**:

```cpp

(process)(t, 42);       // Parentheses suppress ADL
app::process(t, 42);    // Qualified call — no ADL

```

The parentheses trick works because `(process)` is a parenthesized expression, not a simple function name, so ADL rules don't apply.

---

## Self-Assessment

### Q1: Show how `swap(a, b)` finds a type-specific swap via ADL without any `using` declaration

```cpp

#include <iostream>
#include <utility>

namespace physics {
    struct Vec3 {
        double x, y, z;

        // Hidden friend swap — only findable via ADL
        friend void swap(Vec3& a, Vec3& b) noexcept {
            std::cout << "physics::swap(Vec3) called via ADL!\n";
            auto tmp = a;
            a = b;
            b = tmp;
        }
    };
}

int main() {
    physics::Vec3 a{1, 2, 3}, b{4, 5, 6};

    // Unqualified call — ADL searches namespace 'physics' because
    // the arguments are of type physics::Vec3
    swap(a, b);  // Output: physics::swap(Vec3) called via ADL!

    std::cout << "a = {" << a.x << ", " << a.y << ", " << a.z << "}\n";
    std::cout << "b = {" << b.x << ", " << b.y << ", " << b.z << "}\n";
    // a = {4, 5, 6}
    // b = {1, 2, 3}
    return 0;
}

```

**How this works:**

- `swap(a, b)` is an **unqualified call** (no namespace prefix).
- Both arguments have type `physics::Vec3`, so the compiler performs ADL and searches namespace `physics`.
- It finds `friend void swap(Vec3&, Vec3&)` inside `Vec3` through ADL.
- No `using` declaration or `using namespace` is needed — ADL alone is sufficient.
- This is the foundation of how generic algorithms customize behavior per type.

### Q2: Explain how ADL interacts with `friend` function definitions inside a class

A `friend` function defined **inside** a class body (without a matching declaration at namespace scope) is called a **hidden friend**. Such a function:

1. **Is injected into the enclosing namespace** for the purposes of ADL, but is **not visible** to normal unqualified or qualified lookup.
2. **Can only be found through ADL** — meaning at least one argument must have the class type (or a derived type).
3. **Does not participate in implicit conversions** for the first argument in operator overloads, making unintended conversions impossible.

```cpp

#include <iostream>
#include <string>

namespace net {
    struct IPAddress {
        uint32_t addr;

        // Hidden friend — only found via ADL
        friend std::string to_string(const IPAddress& ip) {
            return std::to_string((ip.addr >> 24) & 0xFF) + "." +
                   std::to_string((ip.addr >> 16) & 0xFF) + "." +
                   std::to_string((ip.addr >> 8) & 0xFF) + "." +
                   std::to_string(ip.addr & 0xFF);
        }

        friend bool operator==(const IPAddress& a, const IPAddress& b) {
            return a.addr == b.addr;
        }
    };
}

int main() {
    net::IPAddress ip{0xC0A80001}; // 192.168.0.1
    std::cout << to_string(ip) << "\n"; // ADL finds net::to_string

    // to_string(42);  // ERROR: net::to_string not visible without ADL
    // net::to_string(ip); // ERROR: hidden friend has no namespace-scope declaration

    net::IPAddress a{0x0A000001}, b{0x0A000001};
    if (a == b) std::cout << "Same IP\n"; // ADL finds operator==
    return 0;
}

```

**Key points:**

- Hidden friends reduce overload set pollution — they don't appear as candidates when arguments don't match.
- This is the recommended pattern for operators and customization points in modern C++.
- The C++ standard library increasingly uses hidden friends (e.g., `std::ranges` customization points).

### Q3: Demonstrate a case where ADL picks an unexpected function overload and how to suppress it

```cpp

#include <iostream>

namespace network {
    struct Packet {
        int id;
        int size;
    };

    // A function in the same namespace as Packet
    void send(const Packet& p, int flags) {
        std::cout << "network::send(Packet, flags=" << flags << ")\n";
    }
}

namespace app {
    // app's own version of send
    void send(const network::Packet& p, int priority) {
        std::cout << "app::send(Packet, priority=" << priority << ")\n";
    }

    void process() {
        network::Packet pkt{1, 1024};

        // PROBLEM: unqualified call — ADL adds network::send to the overload set!
        // Both app::send and network::send have identical signatures for this call.
        // This is AMBIGUOUS and won't compile:
        // send(pkt, 5);  // ERROR: ambiguous

        // FIX 1: Use qualified call
        app::send(pkt, 5);       // OK — only app::send

        // FIX 2: Use parentheses to suppress ADL
        (send)(pkt, 5);          // OK — only finds app::send via normal lookup

        // FIX 3: Assign to a function pointer
        auto fn = static_cast<void(*)(const network::Packet&, int)>(send);
        fn(pkt, 5);              // Calls whichever was found by normal lookup
    }
}

int main() {
    app::process();
    return 0;
}

```

**Output:**

```C++

app::send(Packet, priority=5)
app::send(Packet, priority=5)
app::send(Packet, priority=5)

```cpp

**How this works:**

- The unqualified call `send(pkt, 5)` triggers ADL because `pkt` is of type `network::Packet`.
- ADL adds `network::send` to the candidate set alongside `app::send` from normal lookup.
- Since both have identical parameter types `(const network::Packet&, int)`, the call is **ambiguous**.
- **Suppression techniques:**
  - **Qualified call** `app::send(...)` — explicitly selects the desired function.
  - **Parenthesized name** `(send)(...)` — suppresses ADL entirely, only normal lookup applies.
  - **`static_cast` to function pointer** — resolves at the point of the cast.

---

## Additional Examples

### ADL with Template Arguments

```cpp

#include <iostream>
#include <vector>

namespace math {
    struct Matrix { /* ... */ };

    template<typename T>
    void serialize(const std::vector<T>& v) {
        std::cout << "math::serialize<vector<T>>\n";
    }
}

int main() {
    std::vector<math::Matrix> matrices;
    serialize(matrices);  // ADL searches both 'std' (from vector) and 'math' (from Matrix)
                          // Finds math::serialize
}

```

### ADL with Enums

```cpp

namespace color {
    enum class RGB { Red, Green, Blue };

    std::ostream& operator<<(std::ostream& os, RGB c) {
        switch(c) {
            case RGB::Red:   return os << "Red";
            case RGB::Green: return os << "Green";
            case RGB::Blue:  return os << "Blue";
        }
        return os;
    }
}

int main() {
    std::cout << color::RGB::Red << "\n";  // ADL finds color::operator<<
}

```

### `std::ranges` Customization Points and ADL

```cpp

#include <ranges>
#include <vector>
#include <iostream>

namespace custom {
    struct Container {
        std::vector<int> data;

        friend auto begin(Container& c) { return c.data.begin(); }
        friend auto end(Container& c)   { return c.data.end(); }
    };
}

int main() {
    custom::Container c{{1, 2, 3, 4, 5}};
    for (auto& val : c)  // range-based for uses ADL to find begin/end
        std::cout << val << " ";
    // Output: 1 2 3 4 5
}

```

---

## Notes

- ADL is fundamental to making generic code and operator overloading work naturally in C++.
- The `using std::swap; swap(a, b);` idiom is the gold standard for customization points.
- Prefer **hidden friends** for operators and customization functions to limit the overload set.
- When ADL causes problems, suppress it with qualified calls or parenthesized function names.
- C++20's `std::ranges` customization point objects (CPOs) are designed to handle ADL correctly and consistently.
