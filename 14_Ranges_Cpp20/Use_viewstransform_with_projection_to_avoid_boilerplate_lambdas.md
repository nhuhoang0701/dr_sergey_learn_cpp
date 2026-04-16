# Use views::transform with projection to avoid boilerplate lambdas

**Category:** Ranges (C++20)  
**Item:** #220  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges>  

---

## Topic Overview

C++20 ranges algorithms accept an optional **projection** parameter—a callable applied to each element before the algorithm's operation. This eliminates many trivial lambdas.

### Without vs With Projections

```cpp

// WITHOUT projection (C++17)
std::sort(vec.begin(), vec.end(),
    [](const Person& a, const Person& b) { return a.name < b.name; });

// WITH projection (C++20)
std::ranges::sort(vec, {}, &Person::name);  // {} = default comparator

```

### What Can Be a Projection

| Projection type | Example | What it does |
| --- | --- | --- |
| Pointer to member | `&Person::name` | Extracts `name` field |
| Pointer to member function | `&std::string::size` | Calls `.size()` |
| Lambda | `[](const Person& p) { return p.age * 2; }` | Custom transformation |
| `std::identity{}` | (default) | No-op, passes element unchanged |
| Function pointer | `std::abs` | Applies `abs` before comparison |

### How Projections Work Internally

Ranges algorithms invoke the projection via `std::invoke(proj, element)` before passing the result to the comparator/predicate:

```cpp

For ranges::sort(vec, comp, proj):
  comp(std::invoke(proj, a), std::invoke(proj, b))
  → comp(a.name, b.name)  // when proj = &Person::name

```

### Projection vs views::transform

| Feature | Projection | views::transform |
| --- | --- | --- |
| Scope | Inside one algorithm call | Creates a new view |
| Composable | No — per-algorithm only | Yes — pipes into other views |
| Zero overhead | Yes — inlined by compiler | Yes — lazy, no copies |
| Mutates original | Depends on algorithm | No — transforms on the fly |

---

## Self-Assessment

### Q1: Use a member pointer as a projection in `std::ranges::sort(vec, {}, &Person::name)`

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Person {
    std::string name;
    int age;
    double salary;
};

int main() {
    std::vector<Person> people = {
        {"Charlie", 30, 75000},
        {"Alice",   25, 82000},
        {"Bob",     35, 68000},
        {"Diana",   28, 91000},
    };

    // Sort by name (alphabetical)
    std::ranges::sort(people, {}, &Person::name);
    std::cout << "By name: ";
    for (const auto& p : people) std::cout << p.name << ' ';
    std::cout << '\n';

    // Sort by age (ascending)
    std::ranges::sort(people, {}, &Person::age);
    std::cout << "By age:  ";
    for (const auto& p : people) std::cout << p.name << '(' << p.age << ") ";
    std::cout << '\n';

    // Sort by salary (descending)
    std::ranges::sort(people, std::greater{}, &Person::salary);
    std::cout << "By salary (desc): ";
    for (const auto& p : people)
        std::cout << p.name << '(' << p.salary << ") ";
    std::cout << '\n';

    // Find person with max salary
    auto richest = std::ranges::max(people, {}, &Person::salary);
    std::cout << "Highest paid: " << richest.name << '\n';
}
// Expected output:
// By name: Alice Bob Charlie Diana
// By age:  Alice Diana Charlie Bob
// By salary (desc): Diana Alice Charlie Bob
// Highest paid: Diana

```

**How this works:**

- `&Person::name` is a pointer to data member—`std::invoke` applies it to extract the `name` field.
- `{}` for the comparator means `std::ranges::less{}`—the default ascending order.
- `std::greater{}` with `&Person::salary` sorts salaries in descending order.
- `ranges::max` with projection returns the **Person object** (not just the salary) that has the maximum salary.

### Q2: Explain how projections are applied inside range algorithms and their zero-overhead nature

**Projection application mechanism:**

```cpp

// Simplified pseudo-code for ranges::sort with projection:
template<typename R, typename Comp, typename Proj>
void sort(R&& range, Comp comp, Proj proj) {
    // ...
    // When comparing elements a and b:
    if (std::invoke(comp, std::invoke(proj, a), std::invoke(proj, b))) {
        swap(a, b);
    }
    // ...
}

```

**Zero-overhead proof:**

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    int id;
};

int main() {
    std::vector<Employee> team = {
        {"Alice", 3}, {"Bob", 1}, {"Charlie", 2}
    };

    // WITH projection (recommended)
    std::ranges::sort(team, {}, &Employee::id);

    // Equivalent WITHOUT projection (more verbose)
    // std::ranges::sort(team, [](const Employee& a, const Employee& b) {
    //     return a.id < b.id;
    // });

    // Both generate IDENTICAL assembly (zero overhead)
    // The projection &Employee::id is inlined by the compiler to a direct
    // member access, with no function call overhead.

    for (const auto& e : team)
        std::cout << e.name << '(' << e.id << ") ";
    std::cout << '\n';

    // Projection with member function pointer
    std::vector<std::string> words = {"banana", "fig", "apple", "elderberry"};
    std::ranges::sort(words, {}, &std::string::size);  // sort by length
    std::cout << "By length: ";
    for (const auto& w : words) std::cout << w << ' ';
    std::cout << '\n';
}
// Expected output:
// Alice(1) Bob(2) Charlie(3)
// By length: fig apple banana elderberry

```

**Why zero-overhead:**

1. `std::invoke(&Employee::id, emp)` compiles to `emp.id`—no virtual dispatch, no function pointer indirection.
2. The compiler inlines the projection and comparator, producing the same machine code as a hand-written lambda.
3. Projections are constrained by concepts at compile time, so invalid projections produce clear errors.

### Q3: Show that projections can be composed with `views::transform`

```cpp

#include <algorithm>
#include <cmath>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Product {
    std::string name;
    double price;
    int quantity;
};

int main() {
    std::vector<Product> inventory = {
        {"Widget",  9.99,  50},
        {"Gadget", 24.99, 100},
        {"Gizmo",  14.99,  30},
        {"Doohickey", 4.99, 200},
    };

    // views::transform extracts a field (like a projection for views)
    auto names = inventory | std::views::transform(&Product::name);
    std::cout << "Products: ";
    for (const auto& n : names) std::cout << n << ' ';
    std::cout << '\n';

    // Compute total value: transform to price*quantity, then sum
    auto values = inventory | std::views::transform(
        [](const Product& p) { return p.price * p.quantity; });

    double total = std::ranges::fold_left(values, 0.0, std::plus{});
    std::cout << "Total inventory value: $" << std::round(total * 100) / 100 << '\n';

    // Use projection in algorithm ON a transformed view
    // Sort products by total value descending
    std::ranges::sort(inventory, std::greater{},
        [](const Product& p) { return p.price * p.quantity; });

    std::cout << "By total value (desc):\n";
    for (const auto& p : inventory)
        std::cout << "  " << p.name << ": $" << (p.price * p.quantity) << '\n';

    // Pipeline: filter expensive items, project names
    auto expensive_names = inventory
        | std::views::filter([](const Product& p) { return p.price > 10.0; })
        | std::views::transform(&Product::name);

    std::cout << "Expensive: ";
    for (const auto& n : expensive_names) std::cout << n << ' ';
    std::cout << '\n';
}
// Expected output:
// Products: Widget Gadget Gizmo Doohickey
// Total inventory value: $3946
// By total value (desc):
//   Gadget: $2499
//   Doohickey: $998
//   Widget: $499.5
//   Gizmo: $449.7
// Expensive: Gadget Gizmo

```

**Composition insight:**

- `views::transform(&Product::name)` is the **view equivalent** of a projection—it creates a named range of just the names.
- Projections work **inside algorithms** (sort, find, count_if, etc.).
- `views::transform` works **in pipelines** (composable with filter, take, etc.).
- For complex projections (like `price * quantity`), a lambda works in both contexts.

---

## Notes

- Projections are available on **all** ranges algorithms: `sort`, `find`, `count`, `min`, `max`, `copy_if`, etc.
- The default projection is `std::identity{}`—doing nothing.
- `&T::member` works as projection because `std::invoke(ptr_to_member, object)` is defined to extract the member.
- For chaining multiple projections, use `views::transform` instead—projections don't compose directly.
