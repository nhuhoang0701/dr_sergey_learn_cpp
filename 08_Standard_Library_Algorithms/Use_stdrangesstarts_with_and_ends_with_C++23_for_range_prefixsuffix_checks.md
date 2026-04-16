# Use std::ranges::starts_with and ends_with (C++23) for range prefix/suffix checks

**Category:** Standard Library — Algorithms  
**Item:** #236  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/ranges/starts_with>  

---

## Topic Overview

`std::ranges::starts_with` and `std::ranges::ends_with` (C++23) check whether a range begins or ends with a given subsequence. They work on **any forward range**, not just strings.

### Signatures

```cpp

#include <algorithm>

bool std::ranges::starts_with(range, prefix);
bool std::ranges::starts_with(range, prefix, pred, proj1, proj2);

bool std::ranges::ends_with(range, suffix);
bool std::ranges::ends_with(range, suffix, pred, proj1, proj2);

```

### Comparison with String Methods

| Method | Works on | Standard |
| --- | --- | --- |
| `std::string::starts_with("...")` | Strings only | C++20 |
| `std::string::ends_with("...")` | Strings only | C++20 |
| `std::ranges::starts_with(range, prefix)` | Any forward range | C++23 |
| `std::ranges::ends_with(range, suffix)` | Any forward range | C++23 |

---

## Self-Assessment

### Q1: Check if a std::vector<int> starts with a given subsequence using ranges::starts_with

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <string>
#include <array>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // === Basic starts_with ===
    std::vector<int> prefix = {1, 2, 3};
    bool starts = std::ranges::starts_with(data, prefix);
    std::cout << "Starts with {1,2,3}: " << std::boolalpha << starts << "\n";  // true

    std::vector<int> not_prefix = {2, 3, 4};
    std::cout << "Starts with {2,3,4}: " << std::ranges::starts_with(data, not_prefix) << "\n";  // false

    // === Basic ends_with ===
    std::array<int, 3> suffix = {8, 9, 10};
    bool ends = std::ranges::ends_with(data, suffix);
    std::cout << "Ends with {8,9,10}: " << ends << "\n";  // true

    // === Empty prefix/suffix always matches ===
    std::vector<int> empty;
    std::cout << "Starts with empty: " << std::ranges::starts_with(data, empty) << "\n";  // true
    std::cout << "Ends with empty:   " << std::ranges::ends_with(data, empty) << "\n";    // true

    // === Works with string too ===
    std::string text = "Hello, World!";
    std::string hello = "Hello";
    std::cout << "\n\"" << text << "\" starts with \"" << hello << "\": "
              << std::ranges::starts_with(text, hello) << "\n";  // true
    std::cout << "Ends with \"!\": "
              << std::ranges::ends_with(text, std::string("!")) << "\n";  // true

    return 0;
}

```

### Q2: Implement a simple tokenizer that strips known prefixes using starts_with

```cpp

#include <iostream>
#include <string>
#include <string_view>
#include <algorithm>
#include <vector>

struct Token {
    std::string type;
    std::string_view value;
};

// Simple tokenizer: classify strings by known prefixes
Token classify(std::string_view input) {
    struct PrefixRule {
        std::string_view prefix;
        std::string type;
    };

    std::vector<PrefixRule> rules = {
        {"https://", "URL_HTTPS"},
        {"http://",  "URL_HTTP"},
        {"//",       "COMMENT"},
        {"0x",       "HEX_LITERAL"},
        {"0b",       "BIN_LITERAL"},
        {"/*",       "BLOCK_COMMENT"},
    };

    for (const auto& rule : rules) {
        if (std::ranges::starts_with(input, rule.prefix)) {
            return {rule.type, input.substr(rule.prefix.size())};
        }
    }
    return {"UNKNOWN", input};
}

// Strip a prefix if present, return remaining view
std::string_view strip_prefix(std::string_view sv, std::string_view prefix) {
    if (std::ranges::starts_with(sv, prefix))
        return sv.substr(prefix.size());
    return sv;
}

int main() {
    // === Tokenizer ===
    std::vector<std::string_view> inputs = {
        "https://example.com", "http://test.org",
        "// comment here", "0xFF", "0b1010", "hello"
    };

    for (auto input : inputs) {
        auto [type, value] = classify(input);
        std::cout << type << ": '" << value << "'\n";
    }
    // URL_HTTPS: 'example.com'
    // URL_HTTP: 'test.org'
    // COMMENT: ' comment here'
    // HEX_LITERAL: 'FF'
    // BIN_LITERAL: '1010'
    // UNKNOWN: 'hello'

    // === Prefix stripping ===
    std::cout << "\nStripped: '" << strip_prefix("prefix_name", "prefix_") << "'\n";
    // Stripped: 'name'

    // === File extension checking with ends_with ===
    auto is_cpp_source = [](std::string_view filename) {
        return std::ranges::ends_with(filename, std::string_view(".cpp"))
            || std::ranges::ends_with(filename, std::string_view(".cxx"))
            || std::ranges::ends_with(filename, std::string_view(".cc"));
    };

    std::vector<std::string_view> files = {"main.cpp", "header.h", "util.cxx", "test.py"};
    std::cout << "\nC++ sources:\n";
    for (auto f : files)
        if (is_cpp_source(f))
            std::cout << "  " << f << "\n";
    // main.cpp, util.cxx

    return 0;
}

```

### Q3: Show that starts_with and ends_with work on any forward range, not just strings

```cpp

#include <iostream>
#include <vector>
#include <list>
#include <deque>
#include <algorithm>
#include <array>

int main() {
    // === Works with vector ===
    std::vector<int> vec = {10, 20, 30, 40, 50};
    std::cout << "vector starts_with {10,20}: "
              << std::ranges::starts_with(vec, std::vector{10, 20}) << "\n";  // true

    // === Works with list (non-random-access) ===
    std::list<char> lst = {'a', 'b', 'c', 'd', 'e'};
    std::list<char> lst_prefix = {'a', 'b'};
    std::cout << "list starts_with {'a','b'}: "
              << std::ranges::starts_with(lst, lst_prefix) << "\n";  // true

    // === Works with deque ===
    std::deque<double> dq = {1.1, 2.2, 3.3, 4.4};
    std::array<double, 2> dq_suffix = {3.3, 4.4};
    std::cout << "deque ends_with {3.3,4.4}: "
              << std::ranges::ends_with(dq, dq_suffix) << "\n";  // true

    // === Mixed container types ===
    std::vector<int> v = {1, 2, 3, 4, 5};
    std::array<int, 3> a = {1, 2, 3};
    std::cout << "vector starts_with array: "
              << std::ranges::starts_with(v, a) << "\n";  // true

    // === With custom predicate ===
    // Case-insensitive prefix check
    std::string text = "Hello World";
    std::string prefix = "hello";
    bool ci_match = std::ranges::starts_with(text, prefix,
        [](char a, char b) { return std::tolower(a) == std::tolower(b); });
    std::cout << "\nCase-insensitive starts_with: " << ci_match << "\n";  // true

    // === With projection ===
    struct Point { int x, y; };
    std::vector<Point> points = {{1,10}, {2,20}, {3,30}, {4,40}};
    std::vector<int> xs = {1, 2, 3};

    bool x_match = std::ranges::starts_with(points, xs, {}, &Point::x);
    std::cout << "Points x-coords start with {1,2,3}: " << x_match << "\n";  // true

    return 0;
}

```

---

## Notes

- **C++23 required.** Compiler support: GCC 13+, Clang 17+, MSVC 19.37+.
- For **strings specifically**, prefer the C++20 member functions `str.starts_with()` and `str.ends_with()` — they are simpler and available earlier.
- `ends_with` requires the range to be at least a **forward range** (bidirectional for efficiency). For forward-only ranges, it computes sizes first.
- Both algorithms accept **custom predicates** and **projections**, making them powerful for case-insensitive comparisons or comparing struct fields.
- An empty prefix/suffix always matches any range.

```cpp

**How this works:**

- These work on any forward range, not just strings.

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
