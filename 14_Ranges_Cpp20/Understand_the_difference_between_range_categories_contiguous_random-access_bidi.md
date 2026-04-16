# Understand the difference between range categories: contiguous, random-access, bidirectional, forward, input

**Category:** Ranges (C++20)  
**Item:** #256  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges#Range_concepts>  

---

## Topic Overview

C++20 defines a **hierarchy of range concepts** mirroring the iterator categories. Each level adds capabilities:

```cpp

    contiguous_range       ← strongest (elements in contiguous memory)
         ↑
    random_access_range    ← O(1) subscript, advance, distance
         ↑
    bidirectional_range    ← can go backward (--it)
         ↑
    forward_range          ← multi-pass (can iterate multiple times)
         ↑
    input_range            ← weakest (single-pass, read-only)

```

| Category | Example Container | Example View | Key Operation |
| --- | --- | --- | --- |
| `contiguous_range` | `vector`, `array`, `string` | `span` | `data()` pointer |
| `random_access_range` | `deque` | `views::iota(0, 100)` | `r[i]`, `it + n` |
| `bidirectional_range` | `list`, `set`, `map` | `views::reverse` | `--it` |
| `forward_range` | `forward_list` | `views::split` | multi-pass |
| `input_range` | `istream_view` | `views::istream<int>` | single-pass |

---

## Self-Assessment

### Q1: List a container or view example for each of the five range categories

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <deque>
#include <list>
#include <forward_list>
#include <ranges>
#include <concepts>
#include <set>
#include <sstream>

int main() {
    // 1. contiguous_range: elements in contiguous memory
    std::vector<int> vec = {1, 2, 3};
    static_assert(std::ranges::contiguous_range<decltype(vec)>);
    std::cout << "vector: contiguous_range " << std::boolalpha
              << std::ranges::contiguous_range<decltype(vec)> << "\n";

    // 2. random_access_range: O(1) indexing, but NOT necessarily contiguous
    std::deque<int> dq = {1, 2, 3};
    static_assert(std::ranges::random_access_range<decltype(dq)>);
    static_assert(!std::ranges::contiguous_range<decltype(dq)>);
    std::cout << "deque: random_access (not contiguous)\n";

    // 3. bidirectional_range: can go backward
    std::list<int> lst = {1, 2, 3};
    static_assert(std::ranges::bidirectional_range<decltype(lst)>);
    static_assert(!std::ranges::random_access_range<decltype(lst)>);
    std::cout << "list: bidirectional (not random_access)\n";

    // 4. forward_range: multi-pass
    std::forward_list<int> fl = {1, 2, 3};
    static_assert(std::ranges::forward_range<decltype(fl)>);
    static_assert(!std::ranges::bidirectional_range<decltype(fl)>);
    std::cout << "forward_list: forward (not bidirectional)\n";

    // 5. input_range: single-pass
    std::istringstream iss("1 2 3");
    auto istream_rng = std::ranges::istream_view<int>(iss);
    static_assert(std::ranges::input_range<decltype(istream_rng)>);
    static_assert(!std::ranges::forward_range<decltype(istream_rng)>);
    std::cout << "istream_view: input (not forward)\n";

    // Views also have categories:
    auto filtered = vec | std::views::filter([](int x) { return x > 1; });
    static_assert(std::ranges::bidirectional_range<decltype(filtered)>);
    static_assert(!std::ranges::random_access_range<decltype(filtered)>);
    std::cout << "filter_view<vector>: bidirectional (downgraded!)\n";
}
// Expected output:
//   vector: contiguous_range true
//   deque: random_access (not contiguous)
//   list: bidirectional (not random_access)
//   forward_list: forward (not bidirectional)
//   istream_view: input (not forward)
//   filter_view<vector>: bidirectional (downgraded!)

```

---

### Q2: Show why `views::filter` downgrades the range category

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <ranges>
#include <concepts>

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5, 6, 7, 8};

    // vector is contiguous_range
    static_assert(std::ranges::contiguous_range<decltype(v)>);
    std::cout << "vector: contiguous\n";

    // After filter: downgraded to bidirectional!
    auto f = v | std::views::filter([](int x) { return x % 2 == 0; });
    static_assert(std::ranges::bidirectional_range<decltype(f)>);
    static_assert(!std::ranges::random_access_range<decltype(f)>);
    std::cout << "filter(vector): bidirectional (not random_access)\n";

    // WHY? Because filter skips elements.
    // v[3] would mean "the 4th MATCHING element" which requires scanning
    // from the beginning — O(n), not O(1)!
    // So random-access subscript is WRONG for filter.

    // f[0];  // ERROR: no subscript operator
    // f.begin() + 3;  // ERROR: no random-access arithmetic

    // But we CAN go backward (because vector supports --):
    auto it = f.end();
    --it;  // OK: bidirectional
    std::cout << "Last even: " << *it << "\n";

    // transform preserves the category:
    auto t = v | std::views::transform([](int x) { return x * 2; });
    static_assert(std::ranges::random_access_range<decltype(t)>);
    std::cout << "transform(vector): random_access (preserved)\n";
    std::cout << "t[3] = " << t[3] << "\n";  // OK: O(1)

    // Summary of category effects:
    std::cout << "\nView adaptor effects on vector:\n";
    std::cout << "  transform: preserves (random-access)\n";
    std::cout << "  take/drop: preserves (random-access)\n";
    std::cout << "  reverse:   preserves (bidirectional)\n";
    std::cout << "  filter:    downgrades to bidirectional\n";
    std::cout << "  join:      downgrades to input/forward\n";
}
// Expected output:
//   vector: contiguous
//   filter(vector): bidirectional (not random_access)
//   Last even: 8
//   transform(vector): random_access (preserved)
//   t[3] = 8
// ...

```

---

### Q3: Write a constrained algorithm that requires `bidirectional_range`

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <list>
#include <forward_list>
#include <ranges>
#include <concepts>

// Algorithm: check if range is a palindrome
// Requires bidirectional_range because we need to iterate backward
template <std::ranges::bidirectional_range R>
    requires std::equality_comparable<std::ranges::range_value_t<R>>
bool is_palindrome(R&& r) {
    auto first = std::ranges::begin(r);
    auto last  = std::ranges::end(r);

    if (first == last) return true;
    --last;  // needs bidirectional!

    while (first != last) {
        if (*first != *last) return false;
        ++first;
        if (first == last) break;
        --last;
    }
    return true;
}

int main() {
    // Works with vector (contiguous ⊃ bidirectional):
    std::vector<int> v1 = {1, 2, 3, 2, 1};
    std::vector<int> v2 = {1, 2, 3, 4, 5};
    std::cout << "vector {1,2,3,2,1}: " << std::boolalpha << is_palindrome(v1) << "\n";
    std::cout << "vector {1,2,3,4,5}: " << is_palindrome(v2) << "\n";

    // Works with list (bidirectional):
    std::list<char> chars = {'r', 'a', 'c', 'e', 'c', 'a', 'r'};
    std::cout << "list 'racecar': " << is_palindrome(chars) << "\n";

    // Does NOT compile with forward_list (only forward, not bidirectional):
    // std::forward_list<int> fl = {1, 2, 1};
    // is_palindrome(fl);  // ERROR: constraint not satisfied

    // Works with string:
    std::string s = "abcba";
    std::cout << "string 'abcba': " << is_palindrome(s) << "\n";

    // Works with views that are bidirectional:
    auto filtered = v1 | std::views::filter([](int x) { return x != 0; });
    std::cout << "filtered: " << is_palindrome(filtered) << "\n";
}
// Expected output:
//   vector {1,2,3,2,1}: true
//   vector {1,2,3,4,5}: false
//   list 'racecar': true
//   string 'abcba': true
//   filtered: true

```

---

## Notes

- **Refinement hierarchy:** Each category refines the one below: `contiguous ⊃ random_access ⊃ bidirectional ⊃ forward ⊃ input`.
- **`output_range`** is orthogonal — it means elements can be written to.
- **`sized_range`:** Orthogonal concept — means `ranges::size()` is O(1). Not all random-access ranges are sized (e.g., unbounded `iota`).
- **View adaptors** may downgrade the category. Always check with `static_assert`.
- **`common_range`:** A range where `begin()` and `end()` return the same type. Required by some legacy algorithms.
