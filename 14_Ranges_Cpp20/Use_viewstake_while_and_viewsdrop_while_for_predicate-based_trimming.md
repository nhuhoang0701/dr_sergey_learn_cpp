# Use views::take_while and views::drop_while for predicate-based trimming

**Category:** Ranges (C++20)  
**Item:** #497  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/take_while_view>  

---

## Topic Overview

This topic focuses on **trimming** use cases and the critical difference between `take_while`/`drop_while` and `filter`.

### take_while vs filter: The Key Distinction

```cpp

Source:   [2] [4] [7] [6] [8]

take_while(even):  [2] [4]         ← STOPS at 7 (first false), ignores 6 and 8
filter(even):      [2] [4] [6] [8] ← SKIPS odd elements, keeps ALL even ones

```

**`take_while` is a prefix operation** — it defines a stopping point.  
**`filter` is a global operation** — it selects from the entire range.

### Trimming Patterns

| Pattern | Implementation |
| --- | --- |
| Trim leading | `drop_while(pred)` |
| Trim trailing | `reverse \| drop_while(pred) \| reverse` |
| Trim both | `drop_while(pred) \| reverse \| drop_while(pred) \| reverse` |
| Take prefix | `take_while(pred)` |
| Take until sentinel | `take_while(not_sentinel)` |

---

## Self-Assessment

### Q1: Use `take_while` to process elements until a sentinel condition is met

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    // Process packets until end-of-stream marker (value -1)
    std::vector<int> packets = {10, 20, 30, -1, 40, 50};

    auto valid = packets | std::views::take_while(
        [](int p) { return p != -1; });

    std::cout << "Valid packets: ";
    for (int p : valid)
        std::cout << p << ' ';
    std::cout << '\n';

    // Process sorted data until values exceed threshold
    std::vector<double> sorted_data = {0.1, 0.3, 0.5, 0.8, 1.2, 1.5, 2.0};

    auto below_threshold = sorted_data | std::views::take_while(
        [](double x) { return x < 1.0; });

    std::cout << "Below 1.0: ";
    for (double x : below_threshold)
        std::cout << x << ' ';
    std::cout << '\n';

    // With infinite ranges: generate until condition
    auto squares_below_1000 = std::views::iota(1)
        | std::views::transform([](int n) { return n * n; })
        | std::views::take_while([](int sq) { return sq < 1000; });

    std::cout << "Squares < 1000: ";
    for (int sq : squares_below_1000)
        std::cout << sq << ' ';
    std::cout << '\n';
}
// Expected output:
// Valid packets: 10 20 30
// Below 1.0: 0.1 0.3 0.5 0.8
// Squares < 1000: 1 4 9 16 25 36 49 64 81 100 121 144 169 196 225 256 289 324 361 400 441 484 529 576 625 676 729 784 841 900 961

```

**How this works:**

- `take_while(p != -1)` processes packets `{10, 20, 30}` and stops **before** `-1`. The elements `40, 50` after the sentinel are never touched.
- `take_while(x < 1.0)` on sorted data efficiently finds the boundary—no need to scan the whole vector.
- On infinite ranges, `take_while` provides the termination condition, making the infinite pipeline finite.

### Q2: Combine `drop_while` + `take_while` to extract a middle segment

```cpp

#include <cctype>
#include <iostream>
#include <ranges>
#include <string>

int main() {
    // Trim leading and trailing whitespace from a string
    std::string text = "   Hello, World!   ";

    auto is_space = [](char c) { return std::isspace(static_cast<unsigned char>(c)); };

    // Trim leading whitespace
    auto ltrimmed = text | std::views::drop_while(is_space);

    std::cout << "Left-trimmed: [";
    for (char c : ltrimmed) std::cout << c;
    std::cout << "]\n";

    // Full trim: also trim right
    // reverse | drop_while | reverse removes trailing spaces
    auto trimmed = text
        | std::views::drop_while(is_space)
        | std::views::reverse
        | std::views::drop_while(is_space)
        | std::views::reverse;

    std::cout << "Fully trimmed: [";
    for (char c : trimmed) std::cout << c;
    std::cout << "]\n";

    // Extract number from "prefix123suffix"
    std::string mixed = "abc12345xyz";
    auto digits = mixed
        | std::views::drop_while([](char c) { return !std::isdigit(static_cast<unsigned char>(c)); })
        | std::views::take_while([](char c) { return std::isdigit(static_cast<unsigned char>(c)); });

    std::cout << "Extracted digits: ";
    for (char c : digits) std::cout << c;
    std::cout << '\n';
}
// Expected output:
// Left-trimmed: [Hello, World!   ]
// Fully trimmed: [Hello, World!]
// Extracted digits: 12345

```

**How this works:**

- **Left trim:** `drop_while(is_space)` skips spaces from the front.
- **Right trim:** `reverse | drop_while(is_space) | reverse` reverses, drops trailing spaces (now at the front), then reverses back.
- **Digit extraction:** `drop_while(!digit)` skips non-digits, then `take_while(digit)` takes the digit run.
- All operations are lazy and zero-copy (views into the original string).

### Q3: Show the difference between `take_while` (stops at first false) and `filter` (skips all false)

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {2, 4, 7, 6, 8, 3, 10};

    auto is_even = [](int n) { return n % 2 == 0; };

    // take_while: STOPS at first odd number
    auto tw = data | std::views::take_while(is_even);
    std::cout << "take_while(even): ";
    for (int x : tw) std::cout << x << ' ';
    std::cout << '\n';

    // filter: SKIPS odd, keeps ALL even
    auto fi = data | std::views::filter(is_even);
    std::cout << "filter(even):     ";
    for (int x : fi) std::cout << x << ' ';
    std::cout << '\n';

    std::cout << '\n';
    std::cout << "Comparison table:\n";
    std::cout << "Element:     ";
    for (int x : data) std::cout << x << '\t';
    std::cout << '\n';

    std::cout << "take_while:  ";
    int tw_count = 0;
    for (int& x : data) {
        if (!is_even(x)) { std::cout << "STOP\n"; break; }
        std::cout << x << '\t';
        ++tw_count;
    }
    if (tw_count == static_cast<int>(data.size())) std::cout << '\n';

    std::cout << "filter:      ";
    for (int x : data)
        std::cout << (is_even(x) ? std::to_string(x) : "skip") << '\t';
    std::cout << '\n';
}
// Expected output:
// take_while(even): 2 4
// filter(even):     2 4 6 8 10
//
// Comparison table:
// Element:     2	4	7	6	8	3	10
// take_while:  2	4	STOP
// filter:      2	4	skip	6	8	skip	10

```

**Key differences:**

| Behavior | `take_while(pred)` | `filter(pred)` |
| --- | --- | --- |
| First `false` element | **Stops the entire range** | **Skips it**, continues |
| Later `true` elements | **Never seen** | **Included** |
| Use case | Prefix extraction, sentinels | Global selection |
| Iterator category | Same as source | Downgrades to at most bidirectional |
| Performance | May be faster (stops early) | Must scan entire range |

---

## Notes

- `take_while` preserves the source range category. `filter` downgrades random-access to bidirectional.
- `drop_while` caches `begin()` on first access—subsequent calls to `begin()` are O(1).
- For trimming both ends of a string, the `drop_while | reverse | drop_while | reverse` pattern is idiomatic.
- `take_while(pred)` on a sorted range is equivalent to binary-searching for the boundary—but it's O(N) not O(log N). For sorted data, `ranges::lower_bound` + subrange may be more efficient.
