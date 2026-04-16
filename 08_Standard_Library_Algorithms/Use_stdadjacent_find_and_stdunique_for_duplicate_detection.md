# Use std::adjacent_find and std::unique for duplicate detection

**Category:** Standard Library — Algorithms  
**Item:** #246  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/unique>  

---

## Topic Overview

### std::adjacent_find

Finds the **first pair of adjacent equal elements** (or elements satisfying a predicate). Returns an iterator to the first element of the pair, or `end()` if no pair is found.

```cpp

auto it = std::adjacent_find(first, last);         // equality
auto it = std::adjacent_find(first, last, pred);   // custom predicate

```

Use cases: detect consecutive duplicates, find first adjacent pair with a relationship (e.g., out of order, difference > threshold).

### std::unique

Removes **consecutive duplicate** elements by moving unique elements to the front. Returns an iterator to the new logical end. Like `remove`, it does **not** actually erase elements.

```cpp

auto new_end = std::unique(first, last);         // equality
auto new_end = std::unique(first, last, pred);   // custom predicate

```

The elements after `new_end` are in an unspecified state (moved-from).

### Key Insight

Both `adjacent_find` and `unique` only detect/remove **consecutive** duplicates. To handle **all** duplicates, sort first.

| Need | Algorithm | Prerequisite |
| --- | --- | --- |
| Find first duplicate pair | `adjacent_find` | Sort first for global duplicates |
| Remove all duplicates | `sort` + `unique` + `erase` | Sort first |
| Check if any duplicates exist | `adjacent_find` after sort | Sort first |
| C++20 | `std::ranges::unique` | Supports projections |

---

## Self-Assessment

### Q1: Use adjacent_find to detect the first pair of consecutive equal elements

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <cmath>

int main() {
    // === Basic: find first consecutive duplicate ===
    std::vector<int> v = {1, 2, 3, 3, 4, 5, 5, 6};

    auto it = std::adjacent_find(v.begin(), v.end());
    if (it != v.end()) {
        std::cout << "First duplicate pair: " << *it << " at index "
                  << (it - v.begin()) << "\n";
    }
    // Output: First duplicate pair: 3 at index 2

    // === Find ALL adjacent duplicate pairs ===
    auto pos = v.begin();
    while ((pos = std::adjacent_find(pos, v.end())) != v.end()) {
        std::cout << "Duplicate pair at index " << (pos - v.begin())
                  << ": " << *pos << "\n";
        ++pos;  // advance past this pair
    }
    // Duplicate pair at index 2: 3
    // Duplicate pair at index 5: 5

    // === Custom predicate: find first adjacent pair that's out of order ===
    std::vector<int> data = {1, 3, 5, 4, 7, 8};
    auto unsorted = std::adjacent_find(data.begin(), data.end(),
                                       std::greater<>{});
    if (unsorted != data.end()) {
        std::cout << "Sort break: " << *unsorted << " > " << *(unsorted + 1)
                  << " at index " << (unsorted - data.begin()) << "\n";
    }
    // Output: Sort break: 5 > 4 at index 2

    // === Practical: detect value jumps > threshold ===
    std::vector<double> temps = {20.1, 20.3, 20.2, 25.8, 25.9};
    auto jump = std::adjacent_find(temps.begin(), temps.end(),
        [](double a, double b) { return std::abs(a - b) > 3.0; });
    if (jump != temps.end()) {
        std::cout << "Temperature jump: " << *jump << " → " << *(jump + 1) << "\n";
    }
    // Output: Temperature jump: 20.2 → 25.8

    // === Check if sorted (adjacent_find with greater) ===
    std::vector<int> sorted = {1, 2, 3, 4, 5};
    bool is_sorted = std::adjacent_find(sorted.begin(), sorted.end(),
                                        std::greater<>{}) == sorted.end();
    std::cout << "Is sorted: " << std::boolalpha << is_sorted << "\n";  // true

    return 0;
}

```

### Q2: Apply unique + erase to remove consecutive duplicates from a sorted vector

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

int main() {
    // === Remove consecutive duplicates from sorted data ===
    std::vector<int> v = {1, 1, 2, 3, 3, 3, 4, 4, 5};

    // unique moves unique elements to front, returns new logical end
    auto new_end = std::unique(v.begin(), v.end());

    std::cout << "After unique (before erase):\n";
    std::cout << "  Unique part: ";
    for (auto it = v.begin(); it != new_end; ++it)
        std::cout << *it << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    std::cout << "  Full vector (size " << v.size() << "): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5 3 4 4 5  ← tail is garbage (moved-from)

    // Erase the tail
    v.erase(new_end, v.end());
    std::cout << "After erase (size " << v.size() << "): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    // === Full pattern: sort + unique + erase (remove all duplicates) ===
    std::vector<int> unsorted = {5, 3, 1, 3, 5, 2, 1, 4, 3};
    std::sort(unsorted.begin(), unsorted.end());
    unsorted.erase(std::unique(unsorted.begin(), unsorted.end()), unsorted.end());

    std::cout << "\nAfter sort+unique+erase: ";
    for (int x : unsorted) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3 4 5

    // === Custom predicate: remove "close enough" duplicates ===
    std::vector<double> measurements = {1.0, 1.001, 1.002, 2.0, 2.001, 3.0};
    // Treat values within 0.01 of each other as duplicates
    measurements.erase(
        std::unique(measurements.begin(), measurements.end(),
                    [](double a, double b) { return std::abs(a - b) < 0.01; }),
        measurements.end()
    );

    std::cout << "Deduplicated measurements: ";
    for (double x : measurements) std::cout << x << " ";
    std::cout << "\n";
    // 1 2 3

    // === C++20 alternative: std::erase + ranges ===
    // std::vector<int> v2 = {1,1,2,2,3};
    // std::ranges::sort(v2);
    // auto [new_end2, _] = std::ranges::unique(v2);
    // v2.erase(new_end2, v2.end());

    return 0;
}

```

### Q3: Show a bug where unique is applied to an unsorted range and not all duplicates are removed

```cpp

#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    // === BUG: unique on unsorted data ===
    std::vector<int> v = {3, 1, 3, 2, 1, 3};

    std::cout << "Original: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Original: 3 1 3 2 1 3

    // unique only removes CONSECUTIVE duplicates!
    v.erase(std::unique(v.begin(), v.end()), v.end());

    std::cout << "After unique+erase (unsorted): ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // After unique+erase (unsorted): 3 1 3 2 1 3
    // NOTHING was removed! No consecutive duplicates exist.

    // === Another example where SOME but not all are removed ===
    std::vector<int> v2 = {1, 1, 2, 3, 1, 1, 2};
    v2.erase(std::unique(v2.begin(), v2.end()), v2.end());

    std::cout << "Partial removal: ";
    for (int x : v2) std::cout << x << " ";
    std::cout << "\n";
    // Partial removal: 1 2 3 1 2
    // The consecutive 1,1 at positions 0-1 was reduced to one 1.
    // The consecutive 1,1 at positions 4-5 was reduced to one 1.
    // But 1 still appears twice (non-consecutively).

    // === FIX: sort first, then unique ===
    std::vector<int> v3 = {3, 1, 3, 2, 1, 3};
    std::sort(v3.begin(), v3.end());           // {1, 1, 2, 3, 3, 3}
    v3.erase(std::unique(v3.begin(), v3.end()), v3.end());

    std::cout << "Fixed (sorted first): ";
    for (int x : v3) std::cout << x << " ";
    std::cout << "\n";
    // Fixed (sorted first): 1 2 3  ← all duplicates removed

    // === Alternative: use std::set for O(n log n) dedup without modifying order ===
    std::vector<int> v4 = {3, 1, 3, 2, 1, 3};
    std::vector<int> result;
    std::set<int> seen;
    for (int x : v4) {
        if (seen.insert(x).second) {
            result.push_back(x);
        }
    }
    std::cout << "Order-preserving dedup: ";
    for (int x : result) std::cout << x << " ";
    std::cout << "\n";
    // Order-preserving dedup: 3 1 2

    return 0;
}

```

**How the bug manifests:**

- `std::unique` only compares **adjacent** elements. Non-adjacent duplicates are invisible to it.
- On unsorted data, duplicates may be scattered, so `unique` misses them entirely.
- The fix is always: **sort first, then apply unique + erase**.
- If you need to preserve original order, use a `std::set`/`std::unordered_set` as a seen-tracker.

---

## Notes

- **`adjacent_find` complexity:** O(n), exactly `n-1` comparisons.
- **`unique` complexity:** O(n), exactly `n-1` comparisons/moves.
- **`unique` does NOT erase** — like `remove`, it returns the new logical end. Always call `.erase()`.
- **C++20:** `std::ranges::unique` returns a subrange of the "garbage" tail, making the erase step cleaner.
- **For `std::list`:** Use the member function `lst.unique()` instead — it actually erases in O(1) per removal.
- **`adjacent_find` as `is_sorted` check:** `adjacent_find(first, last, greater<>{}) == last` is equivalent to `is_sorted(first, last)`.
