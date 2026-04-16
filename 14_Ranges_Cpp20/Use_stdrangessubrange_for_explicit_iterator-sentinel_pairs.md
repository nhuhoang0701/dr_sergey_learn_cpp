# Use std::ranges::subrange for explicit iterator-sentinel pairs

**Category:** Ranges (C++20)  
**Item:** #395  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/subrange>  

---

## Topic Overview

`std::ranges::subrange` is a lightweight, non-owning view that wraps an **iterator-sentinel** pair into a proper range. It bridges the gap between legacy APIs that return pairs of iterators and the modern ranges world that expects a single range object.

### What subrange Provides

| Feature | Description |
| --- | --- |
| Construction | From `(iterator, sentinel)` or `(iterator, sentinel, size)` |
| Models | `ranges::view`, `ranges::borrowed_range` |
| Sized | If `sized_sentinel_for<S, I>` is satisfied |
| Category | Inherits the iterator category of `I` |

### Class Template Signature

```cpp

template<std::input_or_output_iterator I,
         std::sentinel_for<I> S = I,
         std::ranges::subrange_kind K = /* deduced */>
class subrange : public std::ranges::view_interface<subrange<I, S, K>>;

```

### Key Operations

```cpp

subrange sr(first, last);     // from iterator pair
sr.begin();                   // returns first
sr.end();                     // returns last
sr.size();                    // if sized
sr.advance(n);                // returns subrange with begin advanced by n
auto [b, e] = sr;             // structured bindings (C++17)

```

### When to Use subrange

1. **Legacy API returns two iterators** — wrap them to use with ranges algorithms.
2. **Pass a sub-portion of a container** to `ranges::sort`, `ranges::find`, etc.
3. **Store an iterator-sentinel pair** in a variable for reuse.
4. **Destructure with structured bindings** for cleaner code.

---

## Self-Assessment

### Q1: Construct a subrange from two iterators and pass it to a range algorithm

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {9, 3, 7, 1, 5, 8, 2, 6, 4};

    // Sort only the middle portion [index 2, index 7)
    auto sr = std::ranges::subrange(data.begin() + 2, data.begin() + 7);
    std::ranges::sort(sr);

    std::cout << "After partial sort: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    // Count elements > 5 in the subrange
    auto count = std::ranges::count_if(sr, [](int n) { return n > 5; });
    std::cout << "Elements > 5 in subrange: " << count << '\n';

    // Subrange from first 4 elements
    auto first4 = std::ranges::subrange(data.begin(), data.begin() + 4);
    std::cout << "First 4: ";
    for (int x : first4) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// After partial sort: 9 3 1 2 5 8 7 6 4
// Elements > 5 in subrange: 1
// First 4: 9 3 1 2

```

**How this works:**

- `subrange(data.begin() + 2, data.begin() + 7)` creates a view over 5 elements `{7, 1, 5, 8, 2}`.
- `ranges::sort(sr)` sorts them in-place → `{1, 2, 5, 8, 7}`. The rest of `data` is untouched.
- The subrange satisfies `random_access_range` and `sized_range`, so all algorithms work on it.

### Q2: Show how subrange bridges legacy iterator-pair APIs and range-based algorithms

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <string>
#include <vector>

// Legacy API: returns a pair of iterators (old style)
template<typename It>
std::pair<It, It> find_run_of(It first, It last, int value) {
    auto start = std::find(first, last, value);
    auto end = start;
    while (end != last && *end == value) ++end;
    return {start, end};
}

int main() {
    std::vector<int> data = {1, 1, 3, 3, 3, 5, 5, 7};

    // Legacy API returns a pair of iterators
    auto [first, last] = find_run_of(data.begin(), data.end(), 3);

    // Bridge: wrap in subrange to use with ranges algorithms
    auto run = std::ranges::subrange(first, last);

    std::cout << "Run of 3s has " << std::ranges::distance(run)
              << " elements\n";

    // Now use range-based operations on the subrange
    auto transformed = run | std::views::transform([](int n) { return n * 10; });
    std::cout << "Transformed: ";
    for (int x : transformed) std::cout << x << ' ';
    std::cout << '\n';

    // ranges::equal_range also returns a subrange directly (C++20)
    auto eq = std::ranges::equal_range(data, 5);
    std::cout << "Equal range for 5: ";
    for (int x : eq) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Run of 3s has 3 elements
// Transformed: 30 30 30
// Equal range for 5: 5 5

```

**How this works:**

- `find_run_of` returns a legacy `std::pair<It, It>`.
- Wrapping with `std::ranges::subrange(first, last)` converts it into a proper range.
- Now `run` can be piped through views, passed to ranges algorithms, and used in range-for loops.
- Note: `ranges::equal_range` already returns a `subrange` natively—no manual wrapping needed.

### Q3: Use subrange with structured bindings to destructure `[begin, end)` from a function return

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // ranges::remove returns a subrange of the "removed" tail
    auto removed = std::ranges::remove(data, 5);

    // Destructure the subrange with structured bindings
    auto [tail_begin, tail_end] = removed;

    std::cout << "Valid elements: ";
    for (auto it = data.begin(); it != tail_begin; ++it)
        std::cout << *it << ' ';
    std::cout << '\n';

    // Actually erase the tail
    data.erase(tail_begin, tail_end);
    std::cout << "After erase: ";
    for (int x : data) std::cout << x << ' ';
    std::cout << '\n';

    // Another example: partition returns a subrange of the second partition
    std::vector<int> nums = {1, 2, 3, 4, 5, 6, 7, 8, 9};
    auto [second_begin, second_end] = std::ranges::partition(nums,
        [](int n) { return n % 2 == 0; });

    std::cout << "Odd partition: ";
    for (auto it = second_begin; it != second_end; ++it)
        std::cout << *it << ' ';
    std::cout << '\n';
}
// Expected output:
// Valid elements: 1 2 3 4 6 7 8 9 10
// After erase: 1 2 3 4 6 7 8 9 10
// Odd partition: (odd numbers in unspecified order)

```

**How this works:**

- `ranges::remove` returns a `subrange` representing the "garbage" tail after the logical removal.
- `auto [tail_begin, tail_end] = removed;` destructures the subrange into its two iterators.
- This pattern replaces the old erase-remove idiom: `v.erase(std::remove(v.begin(), v.end(), 5), v.end());`.
- Many ranges algorithms return subranges: `partition`, `remove`, `unique`, `equal_range`, etc.

---

## Notes

- `subrange` is a **borrowed range**—it doesn't own the data, so it's safe to return from functions when the underlying data outlives it.
- Use `subrange(it, sentinel, size_hint)` to provide a size when the sentinel type doesn't support `operator-`.
- `subrange::advance(n)` returns a new subrange with `begin()` advanced by `n` steps—useful for skipping elements.
- `std::ranges::subrange` satisfies `view` (it's cheap to copy—just two iterators/sentinels) and can be used anywhere a view is expected.
