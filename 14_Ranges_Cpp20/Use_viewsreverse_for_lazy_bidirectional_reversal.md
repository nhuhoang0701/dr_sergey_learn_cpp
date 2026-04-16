# Use views::reverse for lazy bidirectional reversal

**Category:** Ranges (C++20)  
**Item:** #496  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/reverse_view>  

---

## Topic Overview

`views::reverse` creates a lazy view that iterates a range in reverse order **without copying** any elements. It simply swaps `begin()` and `end()` and uses reverse iterators.

### Requirements

The source range must be at least **bidirectional** (supports `--it`), because reversing requires moving backward.

| Source category | views::reverse available? |
| --- | --- |
| random_access | Yes — result is random_access |
| bidirectional | Yes — result is bidirectional |
| forward | No — compile error |
| input | No — compile error |

### Double-Reverse Optimization

`views::reverse | views::reverse` cancels out—the library is smart enough to return the original range:

```cpp

auto r = v | views::reverse | views::reverse;  // same as v

```

### Common Patterns

```cpp

v | views::reverse                     → iterate backward
v | views::reverse | views::take(3)    → last 3 elements
v | views::reverse | views::drop(2)    → all except last 2

```

---

## Self-Assessment

### Q1: Apply `views::reverse` to a list and show it iterates in reverse without copying

```cpp

#include <iostream>
#include <list>
#include <ranges>

int main() {
    std::list<int> data = {10, 20, 30, 40, 50};

    // Reverse view — no copies
    auto reversed = data | std::views::reverse;

    std::cout << "Original: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    std::cout << "Reversed: ";
    for (int x : reversed) std::cout << x << ' ';
    std::cout << '\n';

    // Prove no copies: modify through the reversed view
    *std::ranges::begin(reversed) = 999;  // modifies last element of data
    std::cout << "After modifying front of reversed view:\n";
    std::cout << "  Original list: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';
    // last element became 999

    // Size check: sizeof(reversed) is just iterators, not a copy of data
    std::cout << "sizeof(data): " << sizeof(data) << '\n';
    std::cout << "sizeof(view): " << sizeof(reversed) << '\n';
}
// Expected output:
// Original: 10 20 30 40 50
// Reversed: 50 40 30 20 10
// After modifying front of reversed view:
//   Original list: 10 20 30 40 999
// sizeof(data): <implementation-defined>
// sizeof(view): <implementation-defined, typically small>

```

**How this works:**

- `views::reverse` wraps the list's `rbegin()`/`rend()` as a view—no elements are copied.
- Writing through the view modifies the original list, proving it's a non-owning reference.
- `std::list` is bidirectional, meeting the minimum requirement for `views::reverse`.

### Q2: Explain why `views::reverse` requires at least a bidirectional range

**Reversal requires `--it` (decrementing iterators).** Here's why each category can or can't be reversed:

| Category | `--it` supported? | `views::reverse` works? |
| --- | --- | --- |
| **random_access** | Yes | Yes — O(1) begin/end swap |
| **bidirectional** | Yes | Yes — O(1) begin/end swap |
| **forward** | No — only `++it` | **No** — can't go backward |
| **input** | No — single-pass | **No** — can't even re-iterate |

**Proof:**

```cpp

#include <forward_list>
#include <iostream>
#include <list>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> vec = {1, 2, 3};
    std::list<int>   lst = {1, 2, 3};
    // std::forward_list<int> fwd = {1, 2, 3};

    // vector: random_access → reverse is random_access
    auto rv = vec | std::views::reverse;
    static_assert(std::ranges::random_access_range<decltype(rv)>);
    std::cout << "vector reversed: random_access\n";

    // list: bidirectional → reverse is bidirectional
    auto rl = lst | std::views::reverse;
    static_assert(std::ranges::bidirectional_range<decltype(rl)>);
    std::cout << "list reversed: bidirectional\n";

    // forward_list: forward only → DOES NOT COMPILE:
    // auto rf = fwd | std::views::reverse;  // ERROR: not bidirectional
    std::cout << "forward_list: cannot reverse (requires bidirectional)\n";
}
// Expected output:
// vector reversed: random_access
// list reversed: bidirectional
// forward_list: cannot reverse (requires bidirectional)

```

**The fundamental reason:** `reverse_view::begin()` returns `std::make_reverse_iterator(base.end())`, and `reverse_iterator::operator++` calls `--base_iterator`. Without `--`, this is impossible.

### Q3: Combine `reverse` with `take` to efficiently get the last N elements of a range

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {10, 20, 30, 40, 50, 60, 70, 80};

    // Last 3 elements (in original order)
    auto last3 = data | std::views::reverse
                      | std::views::take(3)
                      | std::views::reverse;

    std::cout << "Last 3: ";
    for (int x : last3) std::cout << x << ' ';
    std::cout << '\n';

    // Last 3 elements (in reverse order — skip second reverse)
    auto last3_rev = data | std::views::reverse
                          | std::views::take(3);

    std::cout << "Last 3 (reversed): ";
    for (int x : last3_rev) std::cout << x << ' ';
    std::cout << '\n';

    // All except last 2
    auto except_last2 = data | std::views::reverse
                             | std::views::drop(2)
                             | std::views::reverse;

    std::cout << "Except last 2: ";
    for (int x : except_last2) std::cout << x << ' ';
    std::cout << '\n';

    // Find the last even number
    auto last_even = data
        | std::views::reverse
        | std::views::filter([](int n) { return n % 2 == 0; })
        | std::views::take(1);

    std::cout << "Last even: ";
    for (int x : last_even) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Last 3: 60 70 80
// Last 3 (reversed): 80 70 60
// Except last 2: 10 20 30 40 50 60
// Last even: 80

```

**How this works:**

- `reverse | take(3)` grabs the first 3 from the reversed view—which are the **last 3** of the original.
- Adding another `| reverse` restores original order.
- `reverse | drop(2)` skips the last 2 elements.
- `reverse | filter(...) | take(1)` finds the **last** element matching a predicate, efficiently stopping after one match.
- All operations are lazy—no intermediate vector is created.

---

## Notes

- `views::reverse` on a `common_range` (where `begin` and `end` have the same type) simply uses `std::reverse_iterator`.
- The double-reverse optimization: `views::reverse(views::reverse(r))` returns `r` directly.
- `views::reverse` preserves the range category (random_access stays random_access).
- For getting the last N elements with known size: `views::drop(size - N)` is more direct than `reverse | take(N) | reverse`.
