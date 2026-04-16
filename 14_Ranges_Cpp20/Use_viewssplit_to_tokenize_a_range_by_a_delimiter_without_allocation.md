# Use views::split to tokenize a range by a delimiter without allocation

**Category:** Ranges (C++20)  
**Item:** #392  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/split_view>  

---

## Topic Overview

`views::split` tokenizes a range by a delimiter, producing **subranges** (views) into the original data—**zero heap allocation**. This is a significant improvement over traditional approaches like `std::stringstream` or manual `find`/`substr` loops.

### Why Zero-Allocation Matters

```cpp

Traditional: "a,b,c" → stringstream → getline → creates 3 std::string objects (3 allocations)

views::split: "a,b,c" → 3 subranges pointing into original string (0 allocations)

```

The tokens are **views** (iterator pairs) into the source. No characters are copied.

### Converting Tokens

Since tokens are subranges, you may need to convert them:

| Target type | How to convert |
| --- | --- |
| `std::string_view` | `std::string_view(token.begin(), token.end())` |
| `std::string` | `std::string(token.begin(), token.end())` |
| `int` / `double` | `std::from_chars(...)` on the subrange |
| Keep as view | Just use the subrange directly |

### Working with Non-String Ranges

`views::split` works on **any forward range**, not just strings:

```cpp

vector<int> data = {1, 0, 2, 3, 0, 4};
auto groups = data | views::split(0);
// [[1], [2,3], [4]]

```

---

## Self-Assessment

### Q1: Split a string by ',' using `views::split` and collect substrings without `std::string` allocation

```cpp

#include <iostream>
#include <ranges>
#include <string_view>

int main() {
    std::string_view input = "apple,banana,cherry,date";

    // Split by comma — zero allocation
    std::cout << "Tokens (as string_view, no allocation):\n";
    for (auto token : input | std::views::split(',')) {
        // Construct string_view from the subrange — no heap allocation
        std::string_view sv(token.begin(), token.end());
        std::cout << "  [" << sv << "] length=" << sv.size() << '\n';
    }

    // Use tokens directly in comparisons — no string needed
    std::cout << "\nSearching for 'banana': ";
    for (auto token : input | std::views::split(',')) {
        std::string_view sv(token.begin(), token.end());
        if (sv == "banana") {
            std::cout << "found!\n";
            break;
        }
    }

    // Parse integers without allocation
    std::string_view nums = "10,20,30,40,50";
    int sum = 0;
    for (auto token : nums | std::views::split(',')) {
        std::string_view sv(token.begin(), token.end());
        int val = 0;
        std::from_chars(sv.data(), sv.data() + sv.size(), val);
        sum += val;
    }
    std::cout << "Sum of parsed ints: " << sum << '\n';
}
// Expected output:
// Tokens (as string_view, no allocation):
//   [apple] length=5
//   [banana] length=6
//   [cherry] length=6
//   [date] length=4
//
// Searching for 'banana': found!
// Sum of parsed ints: 150

```

**How this works:**

- Each token is a subrange of iterators into the original `string_view`—no `std::string` is created.
- `std::string_view(token.begin(), token.end())` constructs a view from the subrange in O(1).
- `std::from_chars` parses integers directly from the character subrange—no intermediate string needed.
- The entire split+parse pipeline uses zero heap allocation.

### Q2: Explain why `views::split` returns subranges of the original range, not copies

**Design rationale:**

1. **Views are non-owning by design.** The ranges library philosophy is that views never own data—they provide different "windows" into existing data.

2. **O(1) construction per token.** Each token is just two iterators (begin and end within the source range). Creating a `std::string` copy would be O(N) per token.

3. **Composability.** Subranges can be piped into further views without materialization:

```cpp

#include <algorithm>
#include <iostream>
#include <ranges>
#include <string_view>

int main() {
    std::string_view csv = "Alice:95,Bob:87,Charlie:92";

    // Split, then for each token, split again by ':'
    for (auto record : csv | std::views::split(',')) {
        std::string_view rec(record.begin(), record.end());

        // Second-level split on the token itself — still zero allocation
        auto fields = rec | std::views::split(':');
        auto it = fields.begin();

        std::string_view name((*it).begin(), (*it).end());
        ++it;
        std::string_view score_sv((*it).begin(), (*it).end());

        std::cout << name << " scored " << score_sv << '\n';
    }
}
// Expected output:
// Alice scored 95
// Bob scored 87
// Charlie scored 92

```

**Key insight:** Because tokens are subranges, you can split them again (nested split) without any intermediate `std::string` objects. Every level of splitting works on views into the original string.

### Q3: Compare `views::split` with `std::stringstream` tokenization for performance

| Aspect | `views::split` | `std::stringstream` + `getline` |
| --- | --- | --- |
| **Heap allocations** | 0 (subranges into source) | 1 per token (`std::string`) |
| **Copies** | 0 | Full copy of each token |
| **Lazy** | Yes — tokens found on demand | No — `getline` reads immediately |
| **Works with string_view** | Yes | No — requires `std::string` input |
| **Multi-char delimiter** | Yes — `split("::"sv)` | No — `getline` takes single char |
| **Composable** | Yes — pipes into other views | No — must extract to variables |

```cpp

#include <iostream>
#include <ranges>
#include <sstream>
#include <string>
#include <string_view>

int main() {
    std::string input = "one,two,three,four,five";

    // Approach 1: stringstream (allocates per token)
    std::cout << "stringstream: ";
    std::istringstream ss(input);
    std::string token;
    while (std::getline(ss, token, ',')) {
        std::cout << token << ' ';  // 'token' is a heap-allocated std::string
    }
    std::cout << '\n';

    // Approach 2: views::split (zero allocation)
    std::cout << "views::split: ";
    for (auto part : std::string_view(input) | std::views::split(',')) {
        std::string_view sv(part.begin(), part.end());
        std::cout << sv << ' ';  // 'sv' is a zero-cost view
    }
    std::cout << '\n';

    // Approach 3: Composable pipeline (impossible with stringstream)
    auto long_tokens = std::string_view(input)
        | std::views::split(',')
        | std::views::filter([](auto token) {
              return std::ranges::distance(token) > 3;
          });

    std::cout << "Long tokens:  ";
    for (auto part : long_tokens) {
        std::string_view sv(part.begin(), part.end());
        std::cout << sv << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// stringstream: one two three four five
// views::split: one two three four five
// Long tokens:  three four five

```

**Performance summary:** For tokenizing strings, `views::split` is typically faster because it avoids `std::string` allocation per token. The difference grows with the number of tokens and frequency of calls.

---

## Notes

- When splitting a `std::string` (not `string_view`), ensure the string outlives the split view—the view holds references.
- `views::split` works on any `forward_range`, not just strings: `vector<int>{1,0,2,0,3} | views::split(0)` yields `[[1],[2],[3]]`.
- For regex-based splitting, `views::split` is not sufficient—use `<regex>` or a third-party library.
- C++23 `ranges::to<string>()` makes materializing tokens easier: `token | ranges::to<string>()`.
