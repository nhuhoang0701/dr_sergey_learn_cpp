# Use binary search algorithms on sorted ranges

**Category:** Standard Library — Algorithms  
**Item:** #75  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/lower_bound>  

---

## Topic Overview

The standard library provides four binary search algorithms that work on **sorted ranges**. All have O(log n) complexity for random-access iterators.

### The Four Functions

| Function | Returns | What it finds |
| --- | --- | --- |
| `lower_bound(first, last, val)` | Iterator | First element **≥ val** |
| `upper_bound(first, last, val)` | Iterator | First element **> val** |
| `equal_range(first, last, val)` | Pair of iterators | Range of elements **== val** (lower_bound, upper_bound) |
| `binary_search(first, last, val)` | `bool` | Whether val exists (true/false only) |

### Visual

```cpp

Sorted: {1, 2, 4, 4, 4, 7, 9}
         ↑              ↑     ↑
         |              |     end()
         |              |
  lower_bound(4) ──→ [2]    upper_bound(4) ──→ [5]
  
  equal_range(4) = {[2], [5]}  → elements at indices 2,3,4 are all 4
  binary_search(4) = true
  
  lower_bound(5) ──→ [5]  (first element >= 5 is 7 at index 5)
  upper_bound(5) ──→ [5]  (first element > 5 is 7 at index 5)
  equal_range(5) = {[5], [5]}  → empty range (5 not in container)
  binary_search(5) = false

```

### Prerequisite

All binary search algorithms require the range to be **partitioned** with respect to the value (which a sorted range always satisfies). Calling them on unsorted data is **undefined behavior**.

---

## Self-Assessment

### Q1: Explain the difference between lower_bound, upper_bound, equal_range, and binary_search

```cpp

#include <iostream>
#include <vector>
#include <algorithm>

int main() {
    std::vector<int> v = {1, 2, 4, 4, 4, 7, 9, 12, 15};

    int target = 4;

    // === lower_bound: first element >= target ===
    auto lb = std::lower_bound(v.begin(), v.end(), target);
    std::cout << "lower_bound(4): index " << (lb - v.begin())
              << ", value " << *lb << "\n";
    // index 2, value 4 (first 4)

    // === upper_bound: first element > target ===
    auto ub = std::upper_bound(v.begin(), v.end(), target);
    std::cout << "upper_bound(4): index " << (ub - v.begin())
              << ", value " << *ub << "\n";
    // index 5, value 7 (first element after all 4s)

    // === equal_range: [lower_bound, upper_bound) ===
    auto [er_lb, er_ub] = std::equal_range(v.begin(), v.end(), target);
    std::cout << "equal_range(4): indices [" << (er_lb - v.begin())
              << ", " << (er_ub - v.begin()) << ")\n";
    std::cout << "  Count of 4s: " << std::distance(er_lb, er_ub) << "\n";
    // [2, 5), count = 3

    // === binary_search: just true/false ===
    bool found = std::binary_search(v.begin(), v.end(), target);
    std::cout << "binary_search(4): " << std::boolalpha << found << "\n";
    // true

    // === For a missing value ===
    target = 5;
    lb = std::lower_bound(v.begin(), v.end(), target);
    ub = std::upper_bound(v.begin(), v.end(), target);
    auto [er_lb2, er_ub2] = std::equal_range(v.begin(), v.end(), target);

    std::cout << "\nlower_bound(5): index " << (lb - v.begin()) << " → insert point\n";
    std::cout << "upper_bound(5): index " << (ub - v.begin()) << " → same as lower_bound\n";
    std::cout << "equal_range(5): empty range [" << (er_lb2 - v.begin())
              << ", " << (er_ub2 - v.begin()) << ")\n";
    std::cout << "binary_search(5): " << std::binary_search(v.begin(), v.end(), target) << "\n";
    // lower_bound and upper_bound both point to 7 (index 5)
    // equal_range is empty: [5, 5)
    // binary_search returns false

    // === Key differences ===
    // lower_bound: "Where would I insert to keep sorted order?" (before equal elements)
    // upper_bound: "Where would I insert to keep sorted order?" (after equal elements)
    // equal_range: "Give me all elements equal to this value"
    // binary_search: "Is it there?" (least useful — no position info)

    return 0;
}

```

**How it works:**

- **`lower_bound`** returns the first position where `val` could be inserted without breaking sort order (before existing equal elements). Most versatile — use for "find or insert".
- **`upper_bound`** returns the last such position (after existing equal elements).
- **`equal_range`** combines both — the half-open range `[lb, ub)` contains all elements equal to `val`.
- **`binary_search`** only returns bool. Rarely useful because you usually need the position too.

### Q2: Use lower_bound to find the insertion point for a sorted insert

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <string>

// === Sorted insert using lower_bound ===
template <typename T>
void sorted_insert(std::vector<T>& v, const T& val) {
    auto pos = std::lower_bound(v.begin(), v.end(), val);
    v.insert(pos, val);
    // After insert, v remains sorted
}

int main() {
    std::vector<int> v = {1, 3, 5, 7, 9};

    std::cout << "Initial: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";

    // Insert values maintaining sorted order
    sorted_insert(v, 4);
    sorted_insert(v, 0);
    sorted_insert(v, 10);
    sorted_insert(v, 5);  // duplicate — inserted before existing 5

    std::cout << "After inserts: ";
    for (int x : v) std::cout << x << " ";
    std::cout << "\n";
    // Output: 0 1 3 4 5 5 7 9 10

    // === Practical: sorted event log ===
    struct Event {
        int timestamp;
        std::string name;
    };

    auto cmp = [](const Event& e, int t) { return e.timestamp < t; };

    std::vector<Event> log = {
        {100, "Start"}, {200, "Login"}, {400, "Purchase"}, {500, "Logout"}
    };

    // Insert at correct position
    int new_ts = 300;
    auto pos = std::lower_bound(log.begin(), log.end(), new_ts, cmp);
    log.insert(pos, {new_ts, "Search"});

    std::cout << "\nEvent log:\n";
    for (const auto& e : log)
        std::cout << "  " << e.timestamp << ": " << e.name << "\n";
    // 100: Start
    // 200: Login
    // 300: Search  ← inserted here
    // 400: Purchase
    // 500: Logout

    // === Find-or-insert pattern ===
    auto find_or_insert = [](std::vector<int>& v, int val) -> std::vector<int>::iterator {
        auto it = std::lower_bound(v.begin(), v.end(), val);
        if (it != v.end() && *it == val)
            return it;  // already exists
        return v.insert(it, val);  // insert and return iterator
    };

    std::vector<int> unique_sorted;
    find_or_insert(unique_sorted, 5);
    find_or_insert(unique_sorted, 3);
    find_or_insert(unique_sorted, 5);  // already exists, not inserted
    find_or_insert(unique_sorted, 1);

    std::cout << "\nUnique sorted: ";
    for (int x : unique_sorted) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 3 5

    return 0;
}

```

**How it works:**

- `lower_bound` returns the position where `val` should go to maintain sorted order.
- `vector::insert(pos, val)` inserts at that position, shifting subsequent elements right.
- For the "find or insert" pattern, check if `*it == val` after `lower_bound` to avoid duplicates.
- Total complexity: O(log n) for the search + O(n) for the insert (due to shifting). For frequent sorted inserts, prefer `std::set` or `std::flat_set`.

### Q3: Show a bug where binary_search is called on an unsorted range

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <cassert>

int main() {
    // === BUG: binary_search on unsorted data ===
    std::vector<int> unsorted = {5, 2, 8, 1, 9, 3, 7};

    // This is UNDEFINED BEHAVIOR! The range is not sorted.
    bool found = std::binary_search(unsorted.begin(), unsorted.end(), 3);
    std::cout << "binary_search(3) on unsorted: " << std::boolalpha << found << "\n";
    // May return true OR false — result is meaningless!

    // It might return false even though 3 IS in the vector,
    // because binary search looks at the middle (8), sees 3 < 8,
    // searches the left half {5, 2, 8}, looks at middle (2),
    // sees 3 > 2, searches right half {8}, and doesn't find 3.

    // === Prevention: check with is_sorted ===
    if (!std::is_sorted(unsorted.begin(), unsorted.end())) {
        std::cout << "WARNING: range is not sorted! Binary search invalid.\n";
    }

    // === Fix: sort first ===
    std::sort(unsorted.begin(), unsorted.end());
    found = std::binary_search(unsorted.begin(), unsorted.end(), 3);
    std::cout << "binary_search(3) after sort: " << found << "\n";  // true

    // === Another common bug: wrong comparator ===
    std::vector<int> desc = {9, 7, 5, 3, 1};  // sorted descending

    // BUG: binary_search assumes ascending order by default
    found = std::binary_search(desc.begin(), desc.end(), 5);
    std::cout << "\nbinary_search(5) descending without comp: " << found << "\n";
    // UNDEFINED BEHAVIOR — range is not sorted according to default less<>

    // FIX: provide matching comparator
    found = std::binary_search(desc.begin(), desc.end(), 5, std::greater<>{});
    std::cout << "binary_search(5) descending with greater: " << found << "\n";  // true

    // === Another bug: using lower_bound for "contains" incorrectly ===
    std::vector<int> v = {1, 3, 5, 7, 9};

    auto it = std::lower_bound(v.begin(), v.end(), 4);
    // it points to 5 (first element >= 4), NOT to 4
    // BUG: assuming *it == 4 without checking
    if (it != v.end() /* missing: && *it == 4 */) {
        std::cout << "\nBUG: found " << *it << " when looking for 4\n";
        // Prints "found 5" — false positive!
    }

    // CORRECT way to check if value exists:
    if (it != v.end() && *it == 4) {
        std::cout << "Found 4\n";
    } else {
        std::cout << "4 not found (correct)\n";
    }

    return 0;
}

```

**How it works:**

- Binary search algorithms require their input range to be sorted with respect to the comparator. On unsorted data, the algorithm's decisions about which half to search are wrong — it may skip the element entirely.
- **Common bugs:**
  1. Calling `binary_search`/`lower_bound` on unsorted data → UB, wrong results
  2. Using default comparator on descending-sorted data → UB
  3. Assuming `*lower_bound(...) == target` without checking → false positives
- **Defense:** Use `assert(std::is_sorted(first, last))` during development. Use `std::is_sorted_until` to find where sorting breaks.

---

## Notes

- **`binary_search` is almost never the right choice.** It only returns bool. Use `lower_bound` and check the result — you get position information too.
- **Complexity:** O(log n) for random-access iterators. O(n) for forward iterators (can only advance one step at a time, but still uses O(log n) comparisons).
- **Custom comparators:** All four functions accept a comparator as the last argument. It must match the ordering used to sort the range.
- **C++20 ranges versions:** `std::ranges::lower_bound(v, val)` — no begin/end needed. Also supports projections.
- **On `std::set`/`std::map`:** Use `.lower_bound()` member function instead of `std::lower_bound()` — the member function is O(log n) on the tree, while the free function would be O(n) on non-random-access iterators.
- **`std::partition_point`:** A generalization — finds the partition point for any sorted predicate. `lower_bound` is a special case.
