# Understand SFINAE and Know When to Replace It with Concepts

**Category:** Templates & Generic Programming  
**Item:** #48  
**Standard:** C++11 (SFINAE), C++20 (Concepts)  
**Reference:** <https://en.cppreference.com/w/cpp/language/sfinae>  

---

## Topic Overview

### What Is SFINAE

**S**ubstitution **F**ailure **I**s **N**ot **A**n **E**rror — when the compiler substitutes template arguments and the result is ill-formed, it silently removes that overload from the candidate set instead of producing a hard error.

```cpp

template <typename T>
typename T::value_type get_first(const T& c) { return *c.begin(); }
// If T=int, T::value_type is invalid → SFINAE removes this overload (no error)

template <typename T>
T get_first(T val) { return val; }
// If T=int, this is valid → selected

```

### `std::enable_if` — The Classic SFINAE Tool

```cpp

template <typename T>
std::enable_if_t<std::is_integral_v<T>, T>  // Return type exists only for integral T
double_it(T val) { return val * 2; }

```

### When to Replace SFINAE with Concepts (C++20)

| SFINAE | Concepts |
| --- | --- |
| `enable_if_t<is_integral_v<T>, T>` | `requires std::integral<T>` |
| Hard to read, template noise | Clear, declarative |
| Errors appear deep in substitution | Errors name the unsatisfied concept |
| Cannot express complex constraints easily | Composable with `&&`, `||` |
| Does not participate in subsumption | Supports overload ordering |

**Rule:** In C++20+, prefer concepts. Use SFINAE only in pre-C++20 code.

---

## Self-Assessment

### Q1: Write an `enable_if` SFINAE guard and its equivalent `requires` clause side by side

```cpp

#include <iostream>
#include <type_traits>
#include <concepts>
#include <string>

// === SFINAE approach (C++11/14/17) ===

// Version 1: enable_if in return type
template <typename T>
std::enable_if_t<std::is_arithmetic_v<T>, T>
add_sfinae(T a, T b) {
    return a + b;
}

// Version 2: enable_if as extra template parameter
template <typename T, std::enable_if_t<std::is_arithmetic_v<T>, int> = 0>
T multiply_sfinae(T a, T b) {
    return a * b;
}

// === Concepts approach (C++20) ===

// Version 1: requires clause
template <typename T>
    requires std::is_arithmetic_v<T>
T add_concept(T a, T b) {
    return a + b;
}

// Version 2: named concept
template <typename T>
concept Arithmetic = std::is_arithmetic_v<T>;

template <Arithmetic T>
T multiply_concept(T a, T b) {
    return a * b;
}

// Version 3: abbreviated function template (C++20)
auto divide_concept(std::integral auto a, std::integral auto b) {
    return a / b;
}

int main() {
    std::cout << "=== SFINAE vs Concepts side-by-side ===\n\n";

    // Both work identically at runtime:
    std::cout << "SFINAE  add(3, 4)       = " << add_sfinae(3, 4) << "\n";
    std::cout << "Concept add(3, 4)       = " << add_concept(3, 4) << "\n";

    std::cout << "SFINAE  multiply(3, 4)  = " << multiply_sfinae(3, 4) << "\n";
    std::cout << "Concept multiply(3, 4)  = " << multiply_concept(3, 4) << "\n";

    std::cout << "Concept divide(10, 3)   = " << divide_concept(10, 3) << "\n";

    // Both REJECT non-arithmetic types:
    // add_sfinae(std::string("a"), std::string("b"));   // SFINAE: overload not found
    // add_concept(std::string("a"), std::string("b"));  // Concept: constraint not satisfied

    std::cout << "\n=== Error message comparison ===\n";
    std::cout << "SFINAE error:  'no matching function for call to add_sfinae'\n";
    std::cout << "               (no explanation WHY)\n";
    std::cout << "Concept error: 'constraints not satisfied: is_arithmetic_v<string> is false'\n";
    std::cout << "               (tells you EXACTLY what's wrong)\n";

    return 0;
}

```

### Q2: Explain what 'substitution failure is not an error' means and show a case where it silently selects an overload

```cpp

#include <iostream>
#include <type_traits>
#include <string>
#include <vector>

// === Overload 1: for containers with .size() ===
template <typename T>
auto describe(const T& c)
    -> decltype(c.size(), std::string{})   // SFINAE: valid only if c.size() compiles
{
    return "container with " + std::to_string(c.size()) + " elements";
}

// === Overload 2: for types streamable to ostream ===
template <typename T>
auto describe(const T& val)
    -> decltype(std::to_string(val))       // SFINAE: valid only if to_string(val) compiles
{
    return "value: " + std::to_string(val);
}

// HOW SFINAE WORKS HERE:
//
// describe(42):
//   Overload 1: tries 42.size() → int has no .size() → SUBSTITUTION FAILURE → removed (NOT an error!)
//   Overload 2: tries to_string(42) → valid → SELECTED
//
// describe(vector{1,2,3}):
//   Overload 1: tries v.size() → valid → SELECTED (better match)
//   Overload 2: tries to_string(v) → invalid → SUBSTITUTION FAILURE → removed

// === Dangerous silent selection example ===
template <typename T>
auto compute(T val) -> std::enable_if_t<(sizeof(T) <= 4), int> {
    std::cout << "  (small type path: sizeof=" << sizeof(T) << ")\n";
    return static_cast<int>(val);
}

template <typename T>
auto compute(T val) -> std::enable_if_t<(sizeof(T) > 4), long long> {
    std::cout << "  (large type path: sizeof=" << sizeof(T) << ")\n";
    return static_cast<long long>(val);
}

int main() {
    std::cout << "=== SFINAE silently selects overloads ===\n\n";

    std::cout << "describe(42):          " << describe(42) << "\n";
    std::cout << "describe(vector{...}): " << describe(std::vector{1,2,3}) << "\n";
    std::cout << "describe(string):      " << describe(std::string("hello")) << "\n";

    std::cout << "\n=== Silent path selection based on sizeof ===\n";
    std::cout << "compute(42):    " << compute(42) << "\n";       // small (int=4)
    std::cout << "compute(42LL):  " << compute(42LL) << "\n";     // large (long long=8)
    std::cout << "compute('a'):   " << compute('a') << "\n";      // small (char=1)

    std::cout << "\nKey: SFINAE SILENTLY picks one overload.\n";
    std::cout << "If you're not careful, the wrong overload is selected\n";
    std::cout << "without any compiler warning!\n";

    return 0;
}

```

### Q3: List three readability and performance advantages of Concepts over `enable_if`

```cpp

#include <iostream>
#include <concepts>
#include <type_traits>
#include <string>

// === Advantage 1: READABILITY ===
// SFINAE — cryptic, nested, hard to parse:
template <typename T,
          std::enable_if_t<std::is_integral_v<T> &&
                           !std::is_same_v<T, bool> &&
                           (sizeof(T) >= 4), int> = 0>
void sfinae_func(T) {}

// Concept — reads like English:
template <typename T>
concept LargeInt = std::integral<T> && !std::same_as<T, bool> && (sizeof(T) >= 4);

void concept_func(LargeInt auto) {}

// === Advantage 2: ERROR MESSAGES ===
// SFINAE error: "no matching function for call to 'sfinae_func'"
//   → WHY? No clue — you must mentally trace enable_if conditions
//
// Concept error: "constraints not satisfied: LargeInt<T>"
//   → "because 'is_integral_v<string>' is false"
//   → Compiler tells you EXACTLY which part failed

// === Advantage 3: OVERLOAD ORDERING (Subsumption) ===
template <typename T>
concept Numeric = std::is_arithmetic_v<T>;

template <typename T>
concept Integer = Numeric<T> && std::integral<T>;
// Integer subsumes Numeric → compiler knows Integer is more specific

// With concepts: compiler picks the most constrained overload automatically
template <Numeric T>
std::string classify(T) { return "numeric (float or int)"; }

template <Integer T>
std::string classify(T) { return "integer (most constrained)"; }

// With SFINAE: this would be AMBIGUOUS — enable_if has no subsumption!

int main() {
    std::cout << "=== 3 Advantages of Concepts over SFINAE ===\n\n";

    std::cout << "1. READABILITY\n";
    std::cout << "   SFINAE: enable_if_t<is_integral_v<T> && !is_same_v<T,bool> && ...>\n";
    std::cout << "   Concept: LargeInt auto val\n";
    std::cout << "   → Concepts are self-documenting\n\n";

    std::cout << "2. ERROR MESSAGES\n";
    std::cout << "   SFINAE: 'no matching function' (no explanation)\n";
    std::cout << "   Concept: 'constraint not satisfied: integral<string> is false'\n";
    std::cout << "   → Concepts point to the exact failing condition\n\n";

    std::cout << "3. OVERLOAD ORDERING (Subsumption)\n";
    std::cout << "   classify(42):   " << classify(42) << "\n";      // integer
    std::cout << "   classify(3.14): " << classify(3.14) << "\n";    // numeric
    std::cout << "   → Compiler automatically picks the most constrained match\n";
    std::cout << "   → SFINAE cannot do this — it would be ambiguous!\n\n";

    std::cout << "Bonus advantages:\n";
    std::cout << "   4. Faster compilation (concepts fail early, SFINAE tries all overloads)\n";
    std::cout << "   5. Composable with &&, || operators\n";
    std::cout << "   6. Can be used with abbreviated function templates (auto params)\n";

    return 0;
}

```

---

## Notes

- **SFINAE**: substitution failure in template argument deduction silently removes the overload from candidates.
- `std::enable_if_t<condition, T>` is the classic SFINAE gatekeeper — exists only when condition is true.
- SFINAE patterns: return type, extra default template parameter, function parameter.
- **C++20 Concepts** replace SFINAE with cleaner syntax, better errors, and subsumption-based overload ordering.
- SFINAE remains relevant for pre-C++20 code and for working in `enable_if`-heavy legacy codebases.
- Key difference: SFINAE is a compiler mechanism; concepts are a **language feature** designed for constraint expression.
