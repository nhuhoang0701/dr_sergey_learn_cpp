# Use Concepts to Constrain Return Types, Not Just Parameters

**Category:** Templates & Generic Programming  
**Item:** #267  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/function_template>  

---

## Topic Overview

### Constraining Return Types with Concepts

Most developers learn concepts for constraining **parameters**. But C++20 also lets you constrain **return types** using a **constrained placeholder** (`ConceptName auto`):

```cpp

// Constrained PARAMETER (common):
void process(std::integral auto x);

// Constrained RETURN TYPE (this topic):
std::integral auto getCount();    // return type MUST satisfy std::integral

```

### How Return Type Constraints Work

When you write `std::integral auto getCount()`, the compiler:

1. **Deduces** the return type from the `return` statement (like `auto`)
2. **Checks** that the deduced type satisfies the concept
3. **Rejects** the code at compile time if the constraint fails

```cpp

std::integral auto getCount() {
    return 42;     // OK: int satisfies std::integral
    // return 3.14; // ERROR: double does NOT satisfy std::integral
}

```

### Where You Can Constrain Return Types

| Syntax | Meaning |
| --- | --- |
| `std::integral auto f()` | Free function with constrained return |
| `Sortable auto begin()` | Member function with constrained return |
| `auto f() -> std::integral auto` | Trailing return type (same effect) |
| `template<typename T> Printable auto convert(T)` | Template with constrained return |
| In a concept: `{ expr } -> ConceptName` | Compound requirement constraining result |

---

## Self-Assessment

### Q1: Write a function template where the return type is constrained: `std::integral auto getCount()`

```cpp

#include <iostream>
#include <concepts>
#include <vector>
#include <string>
#include <map>

// === Basic constrained return type ===
std::integral auto getCount() {
    return 42;                // OK: int is integral
    // return 42u;            // OK: unsigned int is integral
    // return 3.14;           // ERROR: double is NOT integral
}

// === Template with constrained return type ===
// The return type is deduced but must satisfy std::integral
template <typename Container>
std::integral auto countElements(const Container& c) {
    return static_cast<int>(c.size());  // size_t → int, both integral
}

// === Custom concept for return type constraint ===
template <typename T>
concept NonNegative = std::integral<T> && std::is_unsigned_v<T>;

template <typename Container>
NonNegative auto safeSize(const Container& c) {
    return c.size();  // size_t is unsigned integral → satisfies NonNegative
}

// === Floating point return constraint ===
std::floating_point auto computeAverage(const std::vector<int>& data) {
    double sum = 0;
    for (int v : data) sum += v;
    return sum / data.size();  // double satisfies std::floating_point
}

// === Return type constraint in a class ===
class Inventory {
    std::map<std::string, int> items_;
public:
    void add(std::string name, int qty) { items_[std::move(name)] += qty; }

    // Return type must be integral
    std::integral auto totalItems() const {
        int total = 0;
        for (const auto& [name, qty] : items_)
            total += qty;
        return total;
    }

    // Return type must be unsigned integral
    std::unsigned_integral auto numCategories() const {
        return items_.size();  // size_t is unsigned integral
    }
};

int main() {
    std::cout << "=== Basic constrained return ===\n";
    auto count = getCount();
    std::cout << "getCount() = " << count << "\n";   // 42

    std::cout << "\n=== Template with constrained return ===\n";
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::cout << "countElements = " << countElements(v) << "\n";  // 5
    std::cout << "safeSize = " << safeSize(v) << "\n";            // 5

    std::cout << "\n=== Floating point return ===\n";
    std::cout << "average = " << computeAverage(v) << "\n";  // 3.0

    std::cout << "\n=== Class methods with constrained returns ===\n";
    Inventory inv;
    inv.add("apple", 10);
    inv.add("banana", 5);
    inv.add("cherry", 3);
    std::cout << "totalItems = " << inv.totalItems() << "\n";        // 18
    std::cout << "numCategories = " << inv.numCategories() << "\n";  // 3

    return 0;
}

```

### Q2: Explain what the compiler checks when the return type is a constrained placeholder

When you write a constrained placeholder return type, the compiler performs these checks:

**Step-by-step compilation:**

```cpp

std::integral auto getCount() {
    return 42;
}

```

1. **Type deduction:** The compiler deduces the return type from `return 42;` → `int`
2. **Concept check:** The compiler evaluates `std::integral<int>` → `true`
3. **If check fails:** Hard compile error with a clear diagnostic

```cpp

std::integral auto getFraction() {
    return 3.14;  // Deduced: double
    // Concept check: std::integral<double> → false
    // ERROR: deduced return type 'double' does not satisfy 'integral'
}

```

**What exactly is checked:**

| Check | Description |
| --- | --- |
| Type deduction | Same rules as plain `auto` — deduced from return expression |
| Concept satisfaction | The deduced type is substituted into the concept |
| All return paths | Every `return` statement must deduce the **same** type |
| Constraint consistency | The single deduced type must satisfy the concept |

**Multiple return statements:**

```cpp

std::integral auto getValue(bool flag) {
    if (flag) return 42;      // deduces int
    else return 100;           // also int — OK
    // else return 3.14;       // ERROR: inconsistent deduction (double vs int)
}

```

**The constraint is on the deduced type, not on the expression:**

```cpp

std::integral auto weird() {
    double x = 3.14;
    return static_cast<int>(x);  // Expression involves double, but return TYPE is int
    // std::integral<int> → OK!
}

```

### Q3: Show how to express 'this function returns something that satisfies a concept' in a concept definition

```cpp

#include <iostream>
#include <concepts>
#include <string>
#include <vector>
#include <ranges>

// === Compound requirements constrain return types in concepts ===

// Syntax: { expression } -> ConceptName;
// Meaning: the type of `expression` must satisfy `ConceptName`

// Concept: T must have a .size() that returns something integral
template <typename T>
concept HasIntegralSize = requires(T t) {
    { t.size() } -> std::integral;  
    // ≡ std::integral<decltype(t.size())> must be true
};

// Concept: T must have .begin() returning something that satisfies input_iterator
template <typename T>
concept Iterable = requires(T t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() }   -> std::sentinel_for<decltype(t.begin())>;
};

// Concept: a factory that returns something convertible_to<std::string>
template <typename F>
concept StringFactory = requires(F f) {
    { f() } -> std::convertible_to<std::string>;
};

// Concept: a callable that returns something satisfying a nested concept
template <typename F, typename Arg>
concept TransformFunc = requires(F f, Arg a) {
    { f(a) } -> std::same_as<Arg>;  // must return same type as input
};

// === Using the concepts ===

void printSize(HasIntegralSize auto const& container) {
    std::cout << "  Size: " << container.size() << "\n";
}

void iterate(Iterable auto const& range) {
    std::cout << "  Elements: ";
    for (const auto& elem : range)
        std::cout << elem << " ";
    std::cout << "\n";
}

std::string describe(StringFactory auto factory) {
    return "Created: " + std::string(factory());
}

auto applyTwice(TransformFunc<int> auto func, int value) {
    return func(func(value));  // apply transformation twice
}

int main() {
    std::cout << "=== HasIntegralSize ===\n";
    std::vector<int> v = {10, 20, 30};
    std::string s = "hello";
    printSize(v);   // Size: 3
    printSize(s);   // Size: 5

    std::cout << "\n=== Iterable ===\n";
    iterate(v);
    iterate(s);

    std::cout << "\n=== StringFactory ===\n";
    auto factory = []() { return std::string("world"); };
    std::cout << "  " << describe(factory) << "\n";

    std::cout << "\n=== TransformFunc ===\n";
    auto doubler = [](int x) -> int { return x * 2; };
    std::cout << "  applyTwice(double, 3) = " << applyTwice(doubler, 3) << "\n";  // 12

    std::cout << "\n=== Compound requirement syntax summary ===\n";
    std::cout << "  { expr }                  — expr must be valid\n";
    std::cout << "  { expr } -> Concept       — result must satisfy Concept\n";
    std::cout << "  { expr } -> same_as<int>  — result must be exactly int\n";
    std::cout << "  { expr } noexcept         — expr must be noexcept\n";

    return 0;
}
// Expected output:
//   Size: 3
//   Size: 5
//   Elements: 10 20 30
//   Elements: h e l l o
//   Created: world
//   applyTwice(double, 3) = 12

```

---

## Notes

- **`ConceptName auto f()`** constrains the return type — the deduced type must satisfy the concept.
- The compiler deduces the return type using standard `auto` rules, then checks the concept.
- All `return` statements must deduce the **same type** (same rule as plain `auto` return).
- In concept definitions, use **compound requirements** `{ expr } -> Concept` to constrain what an expression returns.
- `{ t.size() } -> std::integral` means: `t.size()` must be valid AND its return type must satisfy `std::integral`.
- Return-type constraints make APIs self-documenting: `std::integral auto getCount()` clearly states the contract.
