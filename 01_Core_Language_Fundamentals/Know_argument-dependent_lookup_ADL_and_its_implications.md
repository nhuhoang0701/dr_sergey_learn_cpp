# Know argument-dependent lookup (ADL) and its implications

**Category:** Core Language Fundamentals  
**Standard:** C++98 (fundamental rule, refined through C++20/23)  
**Reference:** <https://en.cppreference.com/w/cpp/language/adl>  

---

## Topic Overview

### What Is ADL

**Argument-Dependent Lookup (ADL)**, also called **Koenig Lookup**, is the rule that lets C++ find a function by looking at the types of the arguments you pass to it.

You usually notice ADL when you write an unqualified call like this:

```cpp
func(x);
```

instead of this:

```cpp
ns::func(x);
```

For the unqualified call, the compiler does not only ask, "What functions named `func` are visible from here?" It also asks, "Do any of the argument types point me toward namespaces where a matching `func` might live?"

So the compiler combines two searches:

1. **Normal unqualified lookup** - the current scope, enclosing scopes, `using` declarations, and similar visible names.
2. **ADL** - the associated namespaces of each argument type.

Those candidates are put into one overload set, and then ordinary overload resolution picks the best match.

That is the whole idea. ADL is not a separate calling mechanism. It is just an extra way for the compiler to gather candidate functions before overload resolution starts.

### Why Does ADL Exist

ADL exists because many C++ operations are meant to feel like they belong to a type, even when the function is not written as a member function.

Without ADL, you would need to write awkward calls like this:

```cpp
std::operator<<(std::cout, "hello");  // Ugly!
```

ADL allows the natural syntax:

```cpp
std::cout << "hello";  // operator<< found via ADL from std::cout's namespace
```

The same thing matters for your own types. If `geom::Point` has an `operator<<` or a `swap`, it is natural to put that function in namespace `geom` and let the compiler find it when a `geom::Point` is involved.

So ADL is a convenience rule, but not a cosmetic one. It is one of the rules that makes operators, generic algorithms, and customization points usable without writing namespace names everywhere.

### Associated Namespaces Rules

For a given type `T`, ADL searches namespaces connected to that type. You can think of these as the places where functions related to the type are likely to live.

| Type of argument | Associated namespaces |
| --- | --- |
| Fundamental type (`int`, `double`) | None (no ADL) |
| Class type `ns::C` | `ns`, plus namespaces of base classes |
| Class template `ns::C<ns2::T>` | `ns` and `ns2` |
| Enumeration `ns::E` | `ns` |
| Pointer to `T` / reference to `T` | Same as `T` |
| Function type `R(A1,A2)` | Union of associated namespaces of `R`, `A1`, `A2` |

The important practical point is that ADL follows types, not variable names. If an argument has type `physics::Vec3`, namespace `physics` becomes relevant. If an argument is just an `int`, ADL has nowhere special to look.

The template case is also worth remembering. A type such as `ns::C<ns2::T>` can bring in both `ns` and `ns2`, which is one reason ADL sometimes reaches farther than people expect.

### Key Rules and Edge Cases

Here are the rules that matter most in day-to-day C++:

1. **ADL only applies to unqualified calls** - `ns::f(x)` never triggers ADL.
2. **ADL does NOT apply if an object, variable, or type with the same name is found** in the local scope.
3. **ADL can find `friend` functions** that are only declared inside a class definition. This is the "hidden friend" idiom.
4. **ADL ignores `using` directives, but respects `using` declarations** when building the ADL candidate set (https://stackoverflow.com/q/27544893).
5. **Template functions** often need a visible declaration for ADL to find them correctly, especially when two-phase lookup is involved.

The second rule surprises people because it means a local name can block the lookup path you thought you were using. ADL is powerful, but it is still part of C++ name lookup, and name lookup has a lot of sharp edges.

### The Hidden Friend Idiom

A function defined as `friend` inside a class body, with no out-of-class declaration, can be found by ADL but not by normal qualified lookup. That is why people call it a **hidden friend**.

This is very common for operators:

```cpp
namespace geom {
    struct Point {
        double x, y;
        // Hidden friend - only callable via ADL
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
    // geom::operator==(a, b); // ERROR - hidden friend has no namespace-scope declaration
}
```

The nice part is that the operator is available exactly when a `geom::Point` is involved. It does not pollute normal lookup for unrelated code.

**Benefits of hidden friends:**

- Reduces the overload set (fewer candidates means less noise for the compiler and the reader)
- Prevents unrelated calls from seeing functions that only make sense for this type
- Keeps type-specific customization close to the type itself

### ADL and `swap` - The Classic Pattern

The most famous practical use of ADL is probably `swap`.

In generic code, you usually want two things at once: use `std::swap` as the fallback, but allow a type to provide a better custom `swap` in its own namespace. That is why this pattern exists:

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
                      // It's NOT ambiguous due to a non-template function (`lib::Widget::swap`) is preferred over a function template specialization (`std::swap`).
                      // Reference: https://en.cppreference.com/cpp/language/overload_resolution#:~:text=F1%20is%20a%20non%2Dtemplate%20function%20while%20F2%20is%20a%20template%20specialization
}
```

The line `using std::swap;` gives you the standard fallback. The unqualified call `swap(a, b)` gives ADL a chance to find a type-specific overload.

If you write `std::swap(a, b)` directly, you have disabled that customization path. Sometimes that is fine, but in generic code it is usually not what you want.

### ADL Pitfall: Unexpected Overload Selection

The same rule that makes C++ convenient can also make it surprising. ADL may add a function from an argument's namespace, and that function may be a better overload than the one you expected.

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

From inside `app::run`, your eye may expect `app::process`. But because the argument is an `evil::Token`, ADL adds `evil::process` too. Then overload resolution chooses the best match, and `int` beats `double` for the literal `42`.

This is the main lesson: ADL does not choose a function by itself. It adds candidates. Overload resolution chooses among all candidates afterward.

### Suppressing ADL

When you do not want ADL, say so with the syntax.

Use **qualified calls** or **parentheses around the function name**:

```cpp
(process)(t, 42);       // Parentheses suppress ADL
app::process(t, 42);    // Qualified call - no ADL
```

The parentheses trick works because `(process)` is a parenthesized expression, not a simple function name, so ADL rules do not apply.

Most of the time, the qualified call is clearer. The parenthesized form is useful to recognize because you may see it in library code or tests that intentionally avoid ADL.

---

## Self-Assessment

### Q1: Show how `swap(a, b)` finds a type-specific swap via ADL without any `using` declaration

Here the interesting part is not the swap implementation. The interesting part is where the compiler finds it.

```cpp
#include <iostream>
#include <utility>

namespace physics {
    struct Vec3 {
        double x, y, z;

        // Hidden friend swap - only findable via ADL
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

    // Unqualified call - ADL searches namespace 'physics' because
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

- `swap(a, b)` is an **unqualified call**. There is no namespace prefix.
- Both arguments have type `physics::Vec3`, so ADL searches namespace `physics`.
- It finds the hidden friend `swap(Vec3&, Vec3&)` declared inside `Vec3`.
- No `using namespace physics` is needed. The argument type is enough to make namespace `physics` relevant.
- This is the basic mechanism behind many type-specific customizations in generic code.

### Q2: Explain how ADL interacts with `friend` function definitions inside a class

A `friend` function defined inside a class body, without a matching namespace-scope declaration, is a **hidden friend**.

The function is associated with the enclosing namespace for ADL, but it is not visible to ordinary lookup. In plain terms: you usually cannot name it as `net::to_string`, but the compiler can still find it when one of the arguments is a `net::IPAddress`.

```cpp
#include <iostream>
#include <string>

namespace net {
    struct IPAddress {
        uint32_t addr;

        // Hidden friend - only found via ADL
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

- Hidden friends are available when the class type participates in the call.
- They do not appear as ordinary namespace functions for unrelated calls.
- This keeps overload sets smaller and usually makes diagnostics less noisy.
- The pattern is especially useful for operators and customization functions.

### Q3: Demonstrate a case where ADL picks an unexpected function overload and how to suppress it

This example shows ADL at its most annoying: it adds a function you did not visually ask for, and now the call is ambiguous.

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

        // PROBLEM: unqualified call - ADL adds network::send to the overload set!
        // Both app::send and network::send have identical signatures for this call.
        // This is AMBIGUOUS and won't compile:
        // send(pkt, 5);  // ERROR: ambiguous

        // FIX 1: Use qualified call
        app::send(pkt, 5);       // OK - only app::send

        // FIX 2: Use parentheses to suppress ADL
        (send)(pkt, 5);          // OK - only finds app::send via normal lookup

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

```text
app::send(Packet, priority=5)
app::send(Packet, priority=5)
app::send(Packet, priority=5)
```

**How this works:**

- The unqualified call `send(pkt, 5)` would trigger ADL because `pkt` is a `network::Packet`.
- ADL adds `network::send` to the candidate set alongside `app::send` from normal lookup.
- Since both have identical parameter types `(const network::Packet&, int)`, the call is **ambiguous**.
- **Suppression techniques:**
  - **Qualified call** `app::send(...)` explicitly selects the desired namespace.
  - **Parenthesized name** `(send)(...)` suppresses ADL and keeps only normal lookup.
  - **`static_cast` to function pointer** resolves the function at the point of the cast.

---

## Additional Examples

### ADL with Template Arguments

Template arguments can contribute associated namespaces too. That is why the namespace of `math::Matrix` can matter even though the outer type is `std::vector`.

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

Enums also connect ADL to their enclosing namespace. That is why an `operator<<` beside the enum can be found naturally.

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

Range-based code also depends on this general idea. If `begin` and `end` are tied to your type, ADL can make the type work naturally with range syntax.

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

- ADL is what lets many non-member functions feel attached to the types they operate on.
- The `using std::swap; swap(a, b);` idiom matters because it gives you both a fallback and ADL customization.
- Hidden friends are a good fit for operators and tightly type-specific helper functions.
- When ADL causes problems, suppress it with a qualified call or a parenthesized function name.
- C++20's `std::ranges` customization point objects (CPOs) are designed to handle ADL more consistently than older ad-hoc customization patterns.
