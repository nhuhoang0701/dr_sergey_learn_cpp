# Use range algorithms instead of iterator-pair algorithms

**Category:** Ranges (C++20)  
**Item:** #117  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges>  

---

## Topic Overview

C++20 provides **`std::ranges::` versions** of every classic `<algorithm>` function. These accept a **range** (any object with `begin()`/`end()`) directly, eliminating the repetitive `.begin(), .end()` pattern and adding **projections** as a first-class parameter.

### Classic vs Ranges Algorithm Comparison

| Classic (C++98–C++17) | Ranges (C++20) |
| --- | --- |
| `std::sort(v.begin(), v.end())` | `std::ranges::sort(v)` |
| `std::find(v.begin(), v.end(), 42)` | `std::ranges::find(v, 42)` |
| `std::count_if(v.begin(), v.end(), pred)` | `std::ranges::count_if(v, pred)` |
| `std::copy(a.begin(), a.end(), back_inserter(b))` | `std::ranges::copy(a, back_inserter(b))` |
| No projection support | `std::ranges::sort(v, {}, &T::member)` |
| Returns iterator | Returns structured result (e.g., `in_out_result`) |

### Key Advantages

1. **Less boilerplate** — no `.begin(), .end()` pairs.
2. **Projections** — sort/find/transform by a member or transformation without writing a lambda.
3. **Concept-constrained** — compile errors are clearer (e.g., "does not satisfy `sortable`").
4. **Structured return types** — `ranges::copy` returns `{in, out}`, `ranges::minmax` returns `{min, max}`.
5. **Dangling protection** — passing a temporary where the result would dangle yields `std::ranges::dangling` instead of a silent bug.

### Projection Parameter

Projection is a callable applied to each element **before** comparison/predicate:

```cpp

ranges::sort(people, std::less{}, &Person::age);
//                   ^^^^^^^^^^   ^^^^^^^^^^^^^
//                   comparator   projection: extracts .age

```

The default projection is `std::identity{}` (a no-op).

---

## Self-Assessment

### Q1: Replace `std::sort(v.begin(), v.end())` with `std::ranges::sort(v)` and show both side by side

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> v1 = {5, 2, 8, 1, 9};
    std::vector<int> v2 = v1;  // copy for comparison

    // Classic (C++98)
    std::sort(v1.begin(), v1.end());

    // Ranges (C++20) — cleaner
    std::ranges::sort(v2);

    // Both produce the same result
    std::cout << "Classic: ";
    for (int x : v1) std::cout << x << ' ';
    std::cout << '\n';

    std::cout << "Ranges:  ";
    for (int x : v2) std::cout << x << ' ';
    std::cout << '\n';

    // Ranges also works with subranges:
    std::vector<int> v3 = {9, 7, 5, 3, 1, 8, 6, 4, 2};
    std::ranges::sort(std::ranges::subrange(v3.begin() + 2, v3.begin() + 7));
    std::cout << "Partial: ";
    for (int x : v3) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Classic: 1 2 5 8 9
// Ranges:  1 2 5 8 9
// Partial: 9 7 1 3 5 8 6 4 2

```

**How this works:**

- `std::ranges::sort(v)` deduces `begin(v)` and `end(v)` internally—no manual iterator pairs.
- Both produce identical results; the ranges version is just less typing and less error-prone (you can't accidentally pass mismatched iterators from different containers).
- You can still sort a sub-range by passing a `subrange` or an iterator+sentinel pair.

### Q2: Show that `ranges::sort` accepts a projection to sort by a struct member

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

struct Employee {
    std::string name;
    int         salary;
    int         age;
};

int main() {
    std::vector<Employee> team = {
        {"Alice",   95000, 30},
        {"Bob",     72000, 45},
        {"Charlie", 88000, 28},
        {"Diana",  105000, 35},
    };

    // Sort by salary ascending (projection = &Employee::salary)
    std::ranges::sort(team, std::less{}, &Employee::salary);
    std::cout << "By salary:\n";
    for (const auto& e : team)
        std::cout << "  " << e.name << " $" << e.salary << '\n';

    // Sort by name descending (projection = &Employee::name)
    std::ranges::sort(team, std::greater{}, &Employee::name);
    std::cout << "By name (desc):\n";
    for (const auto& e : team)
        std::cout << "  " << e.name << '\n';

    // Find youngest employee
    auto youngest = std::ranges::min(team, {}, &Employee::age);
    std::cout << "Youngest: " << youngest.name << " (age " << youngest.age << ")\n";
}
// Expected output:
// By salary:
//   Bob $72000
//   Charlie $88000
//   Alice $95000
//   Diana $105000
// By name (desc):
//   Diana
//   Charlie
//   Bob
//   Alice
// Youngest: Charlie (age 28)

```

**How this works:**

- `&Employee::salary` is a **pointer to data member**—ranges algorithms invoke it as a projection.
- The comparator `std::less{}` (or `std::greater{}`) compares the **projected** values, not the `Employee` objects.
- Without projections, you'd write: `std::sort(team.begin(), team.end(), [](const auto& a, const auto& b){ return a.salary < b.salary; });` — much more verbose.
- `ranges::min` also accepts a projection, returning the element itself (not just the projected value).

### Q3: Explain why ranges algorithms do not have execution policy overloads and what to use instead

**Why no `std::execution` policies on ranges algorithms?**

1. **View laziness is incompatible with parallelism.** Views like `filter_view` are single-pass or have non-trivial increment logic. Parallel algorithms need random-access iterators to partition work, but many views don't provide them.

2. **Projection + parallel = complexity.** Adding projections on top of execution policies creates a large combinatorial surface that the committee chose not to standardize initially.

3. **ADL and customization concerns.** Ranges algorithms use `niebloid` objects (not found via ADL), and integrating them with the parallel TS model would require redesign.

**What to use instead:**

```cpp

#include <algorithm>
#include <execution>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {5, 3, 8, 1, 9, 2, 7, 4, 6};

    // Option 1: Classic algorithm with execution policy
    std::sort(std::execution::par, data.begin(), data.end());
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    // Option 2: Materialize a view, then use classic parallel algorithm
    std::vector<int> source = {10, 3, 7, 1, 8, 5, 2, 9, 4, 6};
    // First filter with ranges (lazy)
    auto filtered = source
        | std::views::filter([](int n) { return n > 4; })
        | std::ranges::to<std::vector>();  // C++23 materialization

    // Then sort in parallel with classic algorithm
    std::sort(std::execution::par, filtered.begin(), filtered.end());
    for (int x : filtered) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// 1 2 3 4 5 6 7 8 9
// 5 6 7 8 9 10

```

**Summary table:**

| Need | Solution |
| --- | --- |
| Parallel sort | `std::sort(std::execution::par, v.begin(), v.end())` |
| Parallel transform | `std::transform(std::execution::par_unseq, ...)` |
| Ranges pipeline → parallel | Materialize with `ranges::to<vector>()`, then use classic parallel algorithm |
| Future | P2408 proposes adding execution policies to ranges algorithms |

---

## Notes

- Ranges algorithms are **niebloids** (function objects, not function templates), preventing unintended ADL.
- Passing a temporary container to a ranges algorithm that returns an iterator produces `std::ranges::dangling` at compile time instead of a silent dangling iterator.
- The `{}` default comparator in `ranges::sort(v, {}, &T::member)` means `std::ranges::less{}` (the default), not `std::less<>`—though they behave the same for most types.
- Most ranges algorithms have both a **range overload** (`ranges::sort(range)`) and an **iterator-sentinel overload** (`ranges::sort(first, last)`).
