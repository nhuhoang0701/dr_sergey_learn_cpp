# Understand range algorithms' structured return types

**Category:** Ranges (C++20)  
**Item:** #498  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/copy>  

---

## Topic Overview

C++20 range algorithms return **named structs** instead of single iterators. These structs contain multiple iterators/values, making it easy to continue processing from where an algorithm stopped.

```cpp

// Old: std::copy returns just the output iterator
auto out = std::copy(v.begin(), v.end(), dest.begin());

// New: ranges::copy returns {in, out}
auto [in, out] = std::ranges::copy(v, dest.begin());
// 'in' = past-the-end of input, 'out' = past-the-end of output

```

### Common Return Types

| Algorithm | Return Type | Fields |
| --- | --- | --- |
| `ranges::copy` | `ranges::in_out_result` | `{in, out}` |
| `ranges::mismatch` | `ranges::in_in_result` | `{in1, in2}` |
| `ranges::min_max` | `ranges::min_max_result` | `{min, max}` |
| `ranges::partition` | `ranges::subrange` | iterator pair |
| `ranges::for_each` | `ranges::in_fun_result` | `{in, fun}` |

---

## Self-Assessment

### Q1: Show that `ranges::copy` returns a `{in, out}` struct, not just the output iterator

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <ranges>

int main() {
    std::vector<int> src = {10, 20, 30, 40, 50};
    std::vector<int> dst(5);

    // ranges::copy returns ranges::in_out_result<InputIt, OutputIt>
    auto result = std::ranges::copy(src, dst.begin());

    // Structured bindings:
    auto [in_iter, out_iter] = result;

    std::cout << "Input consumed: " << (in_iter == src.end()) << "\n";  // true
    std::cout << "Output at end: " << (out_iter == dst.end()) << "\n";  // true

    // The struct has named members:
    std::cout << "result.in == src.end(): " << (result.in == src.end()) << "\n";
    std::cout << "result.out == dst.end(): " << (result.out == dst.end()) << "\n";

    // Compare with old std::copy — returns ONLY output iterator:
    auto old_result = std::copy(src.begin(), src.end(), dst.begin());
    // old_result is just an iterator — no info about input position!

    std::cout << "\nCopied: ";
    for (int x : dst) std::cout << x << " ";
    std::cout << "\n";

    // ranges::copy_if also returns {in, out}:
    std::vector<int> evens;
    auto [in2, out2] = std::ranges::copy_if(src, std::back_inserter(evens),
        [](int x) { return x % 20 == 0; });
    std::cout << "Evens: ";
    for (int x : evens) std::cout << x << " ";
    std::cout << "\n";
}
// Expected output:
//   Input consumed: 1
//   Output at end: 1
//   result.in == src.end(): 1
//   result.out == dst.end(): 1
//
//   Copied: 10 20 30 40 50
//   Evens: 20 40

```

---

### Q2: Use structured bindings to continue iteration after `ranges::copy`

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <ranges>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    std::vector<int> first_half(5);
    std::vector<int> second_half(5);

    // Copy first 5 elements:
    auto [mid, out1] = std::ranges::copy_n(data.begin(), 5, first_half.begin());

    // 'mid' points to element 6 in data — continue from there!
    auto [end, out2] = std::ranges::copy(mid, data.end(), second_half.begin());

    std::cout << "First half: ";
    for (int x : first_half) std::cout << x << " ";
    std::cout << "\nSecond half: ";
    for (int x : second_half) std::cout << x << " ";
    std::cout << "\n";

    // Another example: ranges::mismatch returns {in1, in2}
    std::vector<int> a = {1, 2, 3, 4, 5};
    std::vector<int> b = {1, 2, 9, 4, 5};

    auto [it_a, it_b] = std::ranges::mismatch(a, b);
    std::cout << "\nFirst mismatch at index " << (it_a - a.begin())
              << ": " << *it_a << " vs " << *it_b << "\n";

    // Continue comparing from where mismatch stopped:
    auto [it_a2, it_b2] = std::ranges::mismatch(it_a + 1, a.end(), it_b + 1, b.end());
    std::cout << "Next mismatch: " << (it_a2 == a.end() ? "none" : "found") << "\n";
}
// Expected output:
//   First half: 1 2 3 4 5
//   Second half: 6 7 8 9 10
//
//   First mismatch at index 2: 3 vs 9
//   Next mismatch: none

```

---

### Q3: Explain why the new return types are more informative than old iterator-only returns

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <ranges>

int main() {
    std::vector<int> v = {5, 3, 8, 1, 9, 2, 7};

    // OLD: std::minmax_element returns pair<It, It>
    auto [old_min, old_max] = std::minmax_element(v.begin(), v.end());
    std::cout << "Old: min=" << *old_min << " max=" << *old_max << "\n";

    // NEW: ranges::minmax returns min_max_result{min, max} (values!)
    auto [min_val, max_val] = std::ranges::minmax(v);
    std::cout << "New: min=" << min_val << " max=" << max_val << "\n";

    // ranges::for_each returns {in, fun} — preserves stateful functors
    struct Summer {
        int total = 0;
        void operator()(int x) { total += x; }
    };

    auto [end_it, summer] = std::ranges::for_each(v, Summer{});
    std::cout << "Sum: " << summer.total << "\n";
    // Old for_each: returns the functor but NOT the iterator position

    // ranges::partition returns a subrange of the second partition
    auto second_part = std::ranges::partition(v, [](int x) { return x < 5; });
    std::cout << "Partition point at index: " << (second_part.begin() - v.begin()) << "\n";
    std::cout << "Second partition: ";
    for (int x : second_part) std::cout << x << " ";
    std::cout << "\n";
}
// Expected output:
//   Old: min=1 max=9
//   New: min=1 max=9
//   Sum: 35
//   Partition point at index: 3
//   Second partition: 5 9 8 7  (or similar, order depends on partition)

```

**Why structured returns are better:**

| Old (iterator-pair) | New (named struct) |
| --- | --- |
| `pair<It, It>` — `.first` / `.second` | `{.in1, .in2}` — named fields |
| No info about input consumption | `{.in, .out}` — both positions |
| Stateful functors discarded | `{.in, .fun}` — functor returned |
| `pair<T, T>` is generic | `min_max_result<T>` is self-documenting |

---

## Notes

- **Implicit conversion:** Result types convert to `std::pair` and `std::tuple` for backward compatibility.
- **Structured bindings** (`auto [in, out] = ...`) are the idiomatic way to use these returns.
- **All range algorithms** return structured types — check cppreference for each algorithm's specific return type.
- **`ranges::subrange`** is returned by algorithms that produce a range (like `partition`, `remove`).
