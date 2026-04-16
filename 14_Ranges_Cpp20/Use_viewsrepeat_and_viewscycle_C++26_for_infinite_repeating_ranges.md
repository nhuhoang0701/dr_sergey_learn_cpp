# Use views::repeat and views::cycle (C++26) for infinite repeating ranges

**Category:** Ranges (C++20)  
**Item:** #300  
**Standard:** C++23 / C++26  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/repeat_view>  

---

## Topic Overview

### views::repeat (C++23)

`views::repeat(value)` creates an **infinite** range that yields the same value forever. `views::repeat(value, n)` creates a **bounded** range of `n` copies.

```cpp

views::repeat(42)     →  42, 42, 42, 42, ...  (infinite)
views::repeat(42, 3)  →  42, 42, 42           (bounded)

```

Properties:

- **Random-access** and **sized** (bounded) or random-access and unsized (infinite).
- Stores a **single copy** of the value; dereference yields a `const` reference.
- Models `view` (O(1) copy).

### views::cycle (C++26, P2839)

`views::cycle(range)` repeats a **finite range** infinitely:

```cpp

Source: [A, B, C]
views::cycle →  A, B, C, A, B, C, A, B, C, ...  (infinite)

```

Unlike `repeat` (which repeats a single value), `cycle` repeats an **entire sequence**.

### Comparison

| View | Input | Output | Standard |
| --- | --- | --- | --- |
| `repeat(val)` | Single value | val, val, val, ... | C++23 |
| `repeat(val, n)` | Single value + count | val × n | C++23 |
| `cycle(range)` | Any range | range elements repeated | C++26 |

### Safety with Infinite Ranges

Always truncate before consuming:

```cpp

auto safe = views::repeat(42) | views::take(10);  // OK: finite
// auto bad = views::repeat(42) | ranges::to<vector>();  // BAD: infinite allocation!

```

---

## Self-Assessment

### Q1: Create an infinite range that repeats a value using `views::repeat(42) | views::take(10)`

```cpp

#include <iostream>
#include <ranges>
#include <string>

int main() {
    // Infinite repeat, truncated to 10
    auto tens = std::views::repeat(42) | std::views::take(10);
    std::cout << "Ten 42s: ";
    for (int x : tens)
        std::cout << x << ' ';
    std::cout << '\n';

    // Bounded repeat (equivalent)
    auto bounded = std::views::repeat(42, 10);
    std::cout << "Bounded:  ";
    for (int x : bounded)
        std::cout << x << ' ';
    std::cout << '\n';

    // Repeat with non-trivial types
    auto greetings = std::views::repeat(std::string("hello"), 3);
    std::cout << "Greetings: ";
    for (const auto& s : greetings)
        std::cout << s << ' ';
    std::cout << '\n';

    // Prove it's random-access
    auto r = std::views::repeat(7, 100);
    static_assert(std::ranges::random_access_range<decltype(r)>);
    static_assert(std::ranges::sized_range<decltype(r)>);
    std::cout << "Size: " << std::ranges::size(r) << '\n';
    std::cout << "Element [50]: " << r[50] << '\n';
}
// Expected output:
// Ten 42s: 42 42 42 42 42 42 42 42 42 42
// Bounded:  42 42 42 42 42 42 42 42 42 42
// Greetings: hello hello hello
// Size: 100
// Element [50]: 7

```

**How this works:**

- `views::repeat(42)` creates an infinite range; `take(10)` makes it finite.
- `views::repeat(42, 10)` directly creates a bounded 10-element range.
- The view stores **one copy** of the value. `r[50]` returns a reference to that single stored `7`.
- Bounded `repeat` is random-access and sized—`size()` returns the count, and `operator[]` is O(1).

### Q2: Show how `views::cycle` turns a finite range into an infinite cycling one

```cpp

#include <array>
#include <iostream>
#include <ranges>
#include <string_view>

int main() {
    std::array<std::string_view, 3> colors = {"red", "green", "blue"};

    // Cycle indefinitely, take 10
    auto cycling = std::views::cycle(colors) | std::views::take(10);

    std::cout << "Cycling colors: ";
    for (auto c : cycling)
        std::cout << c << ' ';
    std::cout << '\n';
    // red green blue red green blue red green blue red

    // Cycle integers
    std::array<int, 4> pattern = {1, 2, 3, 4};
    auto repeating = std::views::cycle(pattern) | std::views::take(12);

    std::cout << "Pattern x3: ";
    for (int x : repeating)
        std::cout << x << ' ';
    std::cout << '\n';
    // 1 2 3 4 1 2 3 4 1 2 3 4

    // Cycle with enumerate to get indices
    std::cout << "Indexed: ";
    int i = 0;
    for (auto c : std::views::cycle(colors) | std::views::take(7)) {
        std::cout << '[' << i++ << ']' << c << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// Cycling colors: red green blue red green blue red green blue red
// Pattern x3: 1 2 3 4 1 2 3 4 1 2 3 4
// Indexed: [0]red [1]green [2]blue [3]red [4]green [5]blue [6]red

```

**How this works:**

- `views::cycle(colors)` creates an infinite range that repeats `{"red", "green", "blue"}` forever.
- `take(10)` truncates it to 10 elements.
- Unlike `repeat` (which repeats a single value), `cycle` repeats the entire **sequence** of elements.
- The source range must be a `forward_range` (so it can be re-iterated from the beginning).

### Q3: Use `views::cycle` with `views::zip` to pair a short palette with a long list of items

```cpp

#include <array>
#include <iostream>
#include <ranges>
#include <string>
#include <string_view>
#include <vector>

int main() {
    // Short palette of colors
    std::array<std::string_view, 3> palette = {"red", "green", "blue"};

    // Long list of items
    std::vector<std::string> items = {
        "Apple", "Banana", "Cherry", "Date",
        "Elderberry", "Fig", "Grape"
    };

    // Zip items with cycling colors
    auto colored_items = std::views::zip(
        items,
        std::views::cycle(palette)
    );

    std::cout << "Colored items:\n";
    for (auto [item, color] : colored_items) {
        std::cout << "  " << item << " -> " << color << '\n';
    }

    // Alternative with repeat: assign same color to all
    auto mono = std::views::zip(items, std::views::repeat(std::string_view("gray")));
    std::cout << "\nMonochrome:\n";
    for (auto [item, color] : mono) {
        std::cout << "  " << item << " -> " << color << '\n';
    }
}
// Expected output:
// Colored items:
//   Apple -> red
//   Banana -> green
//   Cherry -> blue
//   Date -> red
//   Elderberry -> green
//   Fig -> blue
//   Grape -> red
//
// Monochrome:
//   Apple -> gray
//   Banana -> gray
//   Cherry -> gray
//   Date -> gray
//   Elderberry -> gray
//   Fig -> gray
//   Grape -> gray

```

**How this works:**

- `views::zip(items, cycle(palette))` pairs each item with the next color from the cycling palette.
- `zip` stops when the **shortest** range is exhausted—since `items` is finite (7 elements) and `cycle(palette)` is infinite, `zip` produces exactly 7 pairs.
- The 3-color palette wraps around: items 4-6 get colors `red, green, blue` again.
- `views::repeat` comparison: if you want the **same** value for every item (not cycling), use `repeat`.

---

## Notes

- `views::repeat` is C++23; `views::cycle` is C++26 (P2839R1).
- Pre-C++26 workaround for `cycle`: use a custom view or simulate with `views::iota(0) | views::transform([&](int i) { return source[i % source.size()]; })`.
- `repeat` stores one value; `cycle` stores (or references) a range. Both produce infinite ranges by default.
- Infinite ranges model `input_range` at minimum. Bounded `repeat` is `random_access_range` + `sized_range`.
- Always pair infinite ranges with `take`, `take_while`, or `zip` (with a finite range) to bound iteration.
