# Use views::concat (C++26) to lazily concatenate multiple ranges

**Category:** Ranges (C++20)  
**Item:** #238  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/concat_view>  

---

## Topic Overview

`views::concat` (C++26, P2542) lazily concatenates **two or more ranges** into a single range, iterating through each in sequence without copying any elements.

### Before concat: The Problem

```cpp

// Eager approach — copies all elements:
std::vector<int> combined;
combined.insert(combined.end(), a.begin(), a.end());
combined.insert(combined.end(), b.begin(), b.end());
combined.insert(combined.end(), c.begin(), c.end());
// Allocates memory, copies every element

```

### With views::concat

```cpp

// Lazy approach — zero copies, zero allocation:
auto combined = std::views::concat(a, b, c);
// Iterates a, then b, then c seamlessly

```

### Iterator Category Rules

The resulting iterator category is the **weakest** (most restrictive) among all input ranges:

| Input ranges | concat iterator category |
| --- | --- |
| All random_access | random_access (if all sized + common) |
| All bidirectional | bidirectional |
| All forward | forward |
| Any input_only | input |
| Mixed (e.g., bidi + forward) | forward (the weakest) |

### Reference Type

The reference type of the concat view is the **common reference** of all input ranges' reference types. All ranges must have a compatible value type.

### Pre-C++26 Alternatives

| Alternative | Limitation |
| --- | --- |
| `views::join` on a range of ranges | Requires ranges stored in a container; heterogeneous types are harder |
| Boost.Range `join` | External dependency |
| Manual iterator adapter | Verbose, error-prone |

---

## Self-Assessment

### Q1: Concatenate three `std::vector<int>`s into a single lazy range using `views::concat`

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> a = {1, 2, 3};
    std::vector<int> b = {10, 20};
    std::vector<int> c = {100, 200, 300, 400};

    // Lazy concatenation — no copies
    auto all = std::views::concat(a, b, c);

    std::cout << "Concatenated: ";
    for (int x : all)
        std::cout << x << ' ';
    std::cout << '\n';

    // Works with range algorithms
    auto sum = std::ranges::fold_left(all, 0, std::plus{});
    std::cout << "Sum: " << sum << '\n';

    // Size is available if all inputs are sized
    std::cout << "Total elements: " << std::ranges::size(all) << '\n';
}
// Expected output:
// Concatenated: 1 2 3 10 20 100 200 300 400
// Sum: 836
// Total elements: 9

```

**How this works:**

- `views::concat(a, b, c)` creates a view that first yields all elements of `a`, then `b`, then `c`.
- No memory is allocated; the view stores references to the original containers.
- Since all three are `vector<int>` (random-access + sized), the concat view is also sized.
- `ranges::fold_left` (C++23) computes the sum across all concatenated elements.

### Q2: Show that `views::concat` avoids copying elements unlike `std::vector::insert`

```cpp

#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    // Using strings to detect copies (moved-from strings become empty)
    std::vector<std::string> a = {"hello", "world"};
    std::vector<std::string> b = {"foo", "bar"};

    // Approach 1: vector::insert (COPIES)
    std::vector<std::string> combined;
    combined.insert(combined.end(), a.begin(), a.end());
    combined.insert(combined.end(), b.begin(), b.end());
    std::cout << "After insert, a[0] = \"" << a[0] << "\" (still valid — was copied)\n";

    // Approach 2: views::concat (NO COPIES)
    auto lazy = std::views::concat(a, b);
    std::cout << "concat view: ";
    for (const auto& s : lazy)
        std::cout << s << ' ';
    std::cout << '\n';

    // Modify through the view — changes the original!
    auto it = std::ranges::begin(lazy);
    *it = "HELLO";  // modifies a[0] directly
    std::cout << "After modifying via view, a[0] = \"" << a[0] << "\"\n";

    // Memory comparison
    std::cout << "combined occupies " << combined.capacity() * sizeof(std::string)
              << " bytes\n";
    std::cout << "concat view occupies ~" << sizeof(lazy) << " bytes (just iterators)\n";
}
// Expected output:
// After insert, a[0] = "hello" (still valid — was copied)
// concat view: hello world foo bar
// After modifying via view, a[0] = "HELLO"
// combined occupies <implementation-defined> bytes
// concat view occupies ~<small number> bytes (just iterators)

```

**How this works:**

- `vector::insert` copies every string into `combined`—if `a` had 1M strings, that's 1M copies.
- `views::concat` stores only references/iterators—O(1) memory regardless of range size.
- Writing through the concat view's iterator modifies the **original** container element.
- This proves the view is non-owning and zero-copy.

### Q3: Explain the iterator category of the concat view based on its input ranges

**The concat view's iterator category is the weakest common category among all input ranges.**

```cpp

#include <concepts>
#include <iostream>
#include <list>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> v = {1, 2, 3};       // random_access_range
    std::list<int>   l = {4, 5, 6};       // bidirectional_range

    // Case 1: All random-access
    auto c1 = std::views::concat(v, v);
    static_assert(std::ranges::random_access_range<decltype(c1)>);
    std::cout << "vector + vector = random_access\n";

    // Case 2: random-access + bidirectional = bidirectional
    auto c2 = std::views::concat(v, l);
    static_assert(std::ranges::bidirectional_range<decltype(c2)>);
    static_assert(!std::ranges::random_access_range<decltype(c2)>);
    std::cout << "vector + list = bidirectional\n";

    // Case 3: bidirectional + bidirectional = bidirectional
    auto c3 = std::views::concat(l, l);
    static_assert(std::ranges::bidirectional_range<decltype(c3)>);
    std::cout << "list + list = bidirectional\n";

    // Case 4: With filter_view (forward only)
    auto fv = v | std::views::filter([](int n) { return n > 1; });
    auto c4 = std::views::concat(v, fv);
    // filter_view is bidirectional when source is random_access,
    // but concat takes the weakest — still bidirectional here
    static_assert(std::ranges::bidirectional_range<decltype(c4)>);
    std::cout << "vector + filter_view = bidirectional\n";
}
// Expected output:
// vector + vector = random_access
// vector + list = bidirectional
// list + list = bidirectional
// vector + filter_view = bidirectional

```

**Category downgrade rules:**

| Rule | Why |
| --- | --- |
| Weakest input wins | concat must switch between ranges at boundaries—it can only guarantee operations supported by ALL inputs |
| Random-access requires all-sized + all-random-access | To compute `it + n`, concat needs to know each range's size to determine which sub-range to jump into |
| Common reference required | `*it` must return the same type regardless of which sub-range the iterator currently points to |

---

## Notes

- `views::concat` is a **C++26** feature (P2542R8). As of 2024, GCC 15 and MSVC have experimental support.
- Pre-C++26 workaround for same-type ranges: store them in a `vector<span<int>>` and use `views::join`.
- `concat` with a single range returns the range itself (degenerate case).
- Unlike `views::join`, `concat` takes its ranges as **individual arguments**, not as a range-of-ranges.
- `concat` is a **borrowed range** if all input ranges are borrowed ranges.
