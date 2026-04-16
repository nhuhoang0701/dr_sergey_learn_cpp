# Use `std::common_type` to Deduce the Common Type of a Set of Types

**Category:** Templates & Generic Programming  
**Item:** #234  
**Standard:** C++11 (improved in C++14/C++23)  
**Reference:** <https://en.cppreference.com/w/cpp/types/common_type>  

---

## Topic Overview

### What Is `std::common_type`

`std::common_type` determines the **common type** that a set of types can all be implicitly converted to:

```cpp

using T = std::common_type_t<int, double>;    // T = double
using U = std::common_type_t<int, long, float>; // U = float

```

### How It Works

The common type is defined by the **ternary operator** (`?:`). For two types `A` and `B`:

```cpp

std::common_type_t<A, B> ≡ std::decay_t<decltype(true ? declval<A>() : declval<B>())>

```

The compiler determines what type `(cond ? a : b)` would produce, then applies `std::decay` to strip references and cv-qualifiers.

### Key Properties

| Property | Behavior |
| --- | --- |
| `common_type_t<int, double>` | `double` (int → double conversion) |
| `common_type_t<int, int>` | `int` |
| `common_type_t<int&, int>` | `int` (decay removes reference) |
| `common_type_t<const int, int>` | `int` (decay removes const) |
| `common_type_t<short, int>` | `int` (short promotes to int) |
| `common_type_t<int, std::string>` | **Compile error** (no common type) |

### Multi-type: Recursive

```cpp

common_type_t<A, B, C> = common_type_t<common_type_t<A, B>, C>

```

---

## Self-Assessment

### Q1: Use `std::common_type_t<int, double>` to implement a type-safe min that mixes `int` and `double`

```cpp

#include <iostream>
#include <type_traits>
#include <string>

// === Type-safe min using common_type ===
template <typename A, typename B>
std::common_type_t<A, B> safe_min(A a, B b) {
    // Return type is the common type — no narrowing
    using Result = std::common_type_t<A, B>;
    return static_cast<Result>(a) < static_cast<Result>(b)
           ? static_cast<Result>(a)
           : static_cast<Result>(b);
}

// === Variadic version ===
template <typename T>
T safe_min_v(T val) { return val; }

template <typename T, typename U, typename... Rest>
auto safe_min_v(T a, U b, Rest... rest) {
    using Common = std::common_type_t<T, U>;
    Common m = (static_cast<Common>(a) < static_cast<Common>(b))
               ? static_cast<Common>(a)
               : static_cast<Common>(b);
    return safe_min_v(m, rest...);
}

// === Type-safe average ===
template <typename... Args>
std::common_type_t<Args...> safe_average(Args... args) {
    using Common = std::common_type_t<Args...>;
    Common sum = (static_cast<Common>(args) + ...);
    return sum / sizeof...(args);
}

int main() {
    std::cout << "=== safe_min(int, double) ===\n";
    auto m1 = safe_min(3, 2.5);         // common_type_t<int,double> = double
    auto m2 = safe_min(3.14, 4);        // double
    auto m3 = safe_min(10, 20);          // int
    std::cout << "min(3, 2.5) = " << m1 << "\n";    // 2.5
    std::cout << "min(3.14, 4) = " << m2 << "\n";   // 3.14
    std::cout << "min(10, 20) = " << m3 << "\n";     // 10

    std::cout << "\n=== Variadic safe_min ===\n";
    auto m4 = safe_min_v(10, 3.5, 7, 1.2);  // operates on mixed types
    std::cout << "min(10, 3.5, 7, 1.2) = " << m4 << "\n";  // 1.2

    std::cout << "\n=== safe_average ===\n";
    auto avg = safe_average(1, 2, 3.0);  // common = double → 2.0
    std::cout << "avg(1, 2, 3.0) = " << avg << "\n";

    std::cout << "\n=== Type verification ===\n";
    using R = std::common_type_t<int, double>;
    std::cout << "common_type<int,double> is double: "
              << std::is_same_v<R, double> << "\n";  // 1

    return 0;
}

```

### Q2: Explain how `common_type` uses the ternary operator's decay rules

The core mechanism of `std::common_type` for two types `A` and `B`:

```cpp

common_type_t<A, B>  ≡  decay_t< decltype(false ? declval<A>() : declval<B>()) >

```

**Step-by-step:**

1. **Ternary operator** `(cond ? a : b)` — the compiler determines the type of this expression
2. The compiler applies **implicit conversions** to find a type both `a` and `b` convert to
3. **`std::decay`** then strips: `const`, `volatile`, references, and array/function → pointer

**Ternary operator rules for type determination:**

```cpp

// Same type → that type
true ? int{} : int{}            // → int

// Arithmetic promotion
true ? int{} : double{}         // → double (int promoted)
true ? short{} : int{}          // → int (short promoted)
true ? float{} : double{}       // → double (float promoted)

// One converts to the other
true ? int{} : long{}           // → long (int → long)

// Pointer types
true ? (Base*){} : (Derived*){} // → Base* (derived → base)

// decay strips qualifiers
const int& a = 1;
int b = 2;
// decltype(true ? a : b) = const int& → but decay makes it int

```

**Example of decay's effect:**

```cpp

#include <iostream>
#include <type_traits>

int main() {
    // Without decay: const int
    // With decay: int
    static_assert(std::is_same_v<std::common_type_t<const int, int>, int>);

    // Without decay: int&
    // With decay: int
    static_assert(std::is_same_v<std::common_type_t<int&, int>, int>);

    // Array decays to pointer
    static_assert(std::is_same_v<std::common_type_t<int[5], int*>, int*>);

    std::cout << "All static_asserts passed\n";
    return 0;
}

```

### Q3: Show a case where `common_type` gives an unexpected result for non-numeric types

```cpp

#include <iostream>
#include <type_traits>
#include <chrono>

// === Surprise 1: Base/Derived pointer decay ===
struct Base { virtual ~Base() = default; };
struct Derived : Base {};

// common_type_t<Base*, Derived*> = Base*
// Seems reasonable, but it DECAYS — you lose const:
// common_type_t<const Base*, Derived*> = const Base*  (OK here)
// But: common_type_t<Base, Derived> is ILL-FORMED! (no implicit conversion between objects)

// === Surprise 2: chrono duration common type ===
// common_type_t<seconds, milliseconds> = milliseconds (common GCD-based type)
// This is a specialization, not ternary operator!

// === Surprise 3: Signed/unsigned surprise ===
// common_type_t<int, unsigned> = unsigned  (!)
// Because: true ? int{} : unsigned{} → unsigned (standard conversion rules)
// This can cause bugs: safe_min(-1, 0u) → HUGE number (unsigned wraparound)

// === Surprise 4: common_type is NOT common_reference ===
// common_type always decays, so you LOSE references
// common_type_t<int&, int&> = int (not int&!)
// Use std::common_reference_t (C++20) if you need references preserved

int main() {
    std::cout << "=== Signed/unsigned surprise ===\n";
    using T = std::common_type_t<int, unsigned>;
    std::cout << "common_type<int, unsigned>: ";
    if constexpr (std::is_same_v<T, unsigned>)
        std::cout << "unsigned\n";  // This prints!

    // Dangerous: -1 becomes a huge unsigned value
    auto result = static_cast<T>(-1) < static_cast<T>(0u);
    std::cout << "-1 < 0u (as common type): " << result << "\n";  // 0 (false!)
    std::cout << "static_cast<unsigned>(-1) = " << static_cast<unsigned>(-1) << "\n";

    std::cout << "\n=== Pointer types ===\n";
    using P = std::common_type_t<Base*, Derived*>;
    std::cout << "common_type<Base*, Derived*> is Base*: "
              << std::is_same_v<P, Base*> << "\n";  // 1

    // common_type_t<Base, Derived> → ill-formed (no implicit conversion)
    // std::common_type_t<Base, Derived> ct; // ERROR!

    std::cout << "\n=== Reference decay ===\n";
    using R = std::common_type_t<int&, int&>;
    std::cout << "common_type<int&, int&> is int (not int&): "
              << std::is_same_v<R, int> << "\n";  // 1

    std::cout << "\n=== Chrono specialization ===\n";
    using namespace std::chrono;
    using D = std::common_type_t<seconds, milliseconds>;
    std::cout << "common_type<seconds, milliseconds> is milliseconds: "
              << std::is_same_v<D, milliseconds> << "\n";  // 1

    std::cout << "\n=== Lessons ===\n";
    std::cout << "1. signed + unsigned → unsigned (watch for negative values!)\n";
    std::cout << "2. Object types need implicit conversion (Base↔Derived fails)\n";
    std::cout << "3. References are always decayed away\n";
    std::cout << "4. Library types may specialize common_type (chrono does)\n";

    return 0;
}

```

---

## Notes

- `std::common_type_t<A, B>` finds a type both `A` and `B` can convert to, based on ternary operator rules.
- Always applies `std::decay` — strips `const`, `volatile`, and references from the result.
- **Pitfall**: `common_type_t<int, unsigned>` = `unsigned` — mixing signed/unsigned is dangerous!
- For 3+ types, it's recursive: `common_type<A,B,C>` = `common_type<common_type<A,B>, C>`.
- Use `std::common_reference_t` (C++20) when you need to preserve references.
- Library types can specialize `common_type` (e.g., `std::chrono::duration`).
