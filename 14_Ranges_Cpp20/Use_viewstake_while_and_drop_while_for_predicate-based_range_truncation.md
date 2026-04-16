# Use views::take_while and drop_while for predicate-based range truncation

**Category:** Ranges (C++20)  
**Item:** #391  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/take_while_view>  

---

## Topic Overview

`views::take_while(pred)` yields elements from the front **while** `pred` is true, stopping at the first `false`. `views::drop_while(pred)` skips elements **while** `pred` is true, yielding everything after.

### Visual Comparison

```cpp

Source:       [1] [2] [3] [7] [4] [5] [2]
                           ^
                           first element where (x < 5) is false

take_while(x < 5):  [1] [2] [3]              ← stops BEFORE 7
drop_while(x < 5):              [7] [4] [5] [2]  ← starts FROM 7

Note: take_while + drop_while = entire range (non-overlapping split)

```

### Key Properties

| View | Stops when | Includes the boundary element? |
| --- | --- | --- |
| `take_while(pred)` | `pred` returns `false` | No — the first failing element is excluded |
| `drop_while(pred)` | `pred` returns `false` | Yes — the first failing element starts the output |

### Comparison with take/drop (count-based)

| Count-based | Predicate-based |
| --- | --- |
| `take(5)` — first 5 elements | `take_while(pred)` — elements while pred is true |
| `drop(3)` — skip first 3 | `drop_while(pred)` — skip while pred is true |

---

## Self-Assessment

### Q1: Use `take_while` to read elements until a sentinel condition is met

```cpp

#include <iostream>
#include <ranges>
#include <string>
#include <vector>

int main() {
    // Simulate reading lines until an empty line
    std::vector<std::string> lines = {
        "Header: Content-Type",
        "Header: Accept",
        "Header: Auth",
        "",                      // empty line = end of headers
        "Body content here",
        "More body",
    };

    // Take lines while they are non-empty
    auto headers = lines | std::views::take_while(
        [](const std::string& line) { return !line.empty(); });

    std::cout << "Headers:\n";
    for (const auto& h : headers)
        std::cout << "  " << h << '\n';

    // Numeric example: take sorted prefix
    std::vector<int> data = {1, 3, 5, 7, 2, 4, 6};
    auto sorted_prefix = data | std::views::take_while(
        [prev = 0](int x) mutable {
            bool ascending = (x >= prev);
            prev = x;
            return ascending;
        });

    std::cout << "Sorted prefix: ";
    for (int x : sorted_prefix)
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Headers:
//   Header: Content-Type
//   Header: Accept
//   Header: Auth
// Sorted prefix: 1 3 5 7

```

**How this works:**

- `take_while(!empty)` yields headers until the first empty line—the empty line and everything after are excluded.
- The sorted-prefix example uses a **mutable lambda** with captured state to detect where the ascending sequence breaks.
- `take_while` is lazy: it evaluates the predicate one element at a time and stops immediately.

### Q2: Use `drop_while` to skip leading whitespace tokens in a tokenized range

```cpp

#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main() {
    // Skip leading whitespace-only tokens
    std::vector<std::string> tokens = {
        "  ", "\t", "", "Hello", "World", " ", "End"
    };

    auto is_whitespace = [](const std::string& s) {
        return s.empty() || s.find_first_not_of(" \t\n") == std::string::npos;
    };

    auto content = tokens | std::views::drop_while(is_whitespace);

    std::cout << "After skipping whitespace tokens: ";
    for (const auto& t : content)
        std::cout << '[' << t << ']' << ' ';
    std::cout << '\n';

    // Numeric: skip leading zeros
    std::vector<int> digits = {0, 0, 0, 4, 2, 0, 7};
    auto significant = digits | std::views::drop_while(
        [](int d) { return d == 0; });

    std::cout << "After dropping leading zeros: ";
    for (int d : significant)
        std::cout << d;
    std::cout << '\n';
}
// Expected output:
// After skipping whitespace tokens: [Hello] [World] [ ] [End]
// After dropping leading zeros: 4207

```

**How this works:**

- `drop_while(is_whitespace)` skips `"  "`, `"\t"`, `""` and starts from `"Hello"`.
- Elements after the first non-matching element are **all included**, even if they would match the predicate (` ` token appears in output).
- `drop_while` evaluates the predicate only on the leading elements; once it finds a non-match, it includes everything from that point.

### Q3: Combine take_while and drop_while to extract a middle segment matching a pattern

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 10, 11, 12, 13, 4, 5, 6};
    //                                ^^^^^^^^^^^^^^^^
    //                                values >= 10 (the "middle segment")

    // Extract the run of values >= 10
    auto middle = data
        | std::views::drop_while([](int n) { return n < 10; })   // skip before
        | std::views::take_while([](int n) { return n >= 10; });  // take until end

    std::cout << "Middle segment (>= 10): ";
    for (int x : middle)
        std::cout << x << ' ';
    std::cout << '\n';

    // General pattern: extract between two markers
    std::vector<std::string> log = {
        "info: startup",
        "info: ready",
        "ERROR: disk full",
        "ERROR: retry failed",
        "ERROR: giving up",
        "info: shutdown",
    };

    auto errors = log
        | std::views::drop_while([](const std::string& s) {
              return s.substr(0, 5) != "ERROR";
          })
        | std::views::take_while([](const std::string& s) {
              return s.substr(0, 5) == "ERROR";
          });

    std::cout << "Error block:\n";
    for (const auto& line : errors)
        std::cout << "  " << line << '\n';

    // Equivalence: drop_while + take_while partitions into 3 segments
    // [before predicate] [matching segment] [after predicate]
    std::cout << "\nBefore: ";
    for (int x : data | std::views::take_while([](int n) { return n < 10; }))
        std::cout << x << ' ';
    std::cout << "\nMiddle: ";
    for (int x : middle)
        std::cout << x << ' ';
    std::cout << "\nAfter:  ";
    auto after = data
        | std::views::drop_while([](int n) { return n < 10; })
        | std::views::drop_while([](int n) { return n >= 10; });
    for (int x : after)
        std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Middle segment (>= 10): 10 11 12 13
// Error block:
//   ERROR: disk full
//   ERROR: retry failed
//   ERROR: giving up
//
// Before: 1 2 3
// Middle: 10 11 12 13
// After:  4 5 6

```

**How this works:**

- `drop_while(x < 10)` skips `{1, 2, 3}` and starts from `10`.
- `take_while(x >= 10)` then takes `{10, 11, 12, 13}` and stops before `4`.
- The combination extracts the **first contiguous run** matching the pattern.
- The three-segment decomposition (before/middle/after) shows how drop_while + take_while partition a range.

---

## Notes

- `take_while` + `drop_while` are complementary: together they split a range exactly where the predicate first fails.
- `drop_while` caches `begin()` after the first call (O(N) first time, O(1) subsequently).
- For extracting **all** matching elements (not just a contiguous prefix), use `views::filter` instead.
- `take_while` and `drop_while` work with infinite ranges: `iota(0) | take_while(x < 100)` is finite.
