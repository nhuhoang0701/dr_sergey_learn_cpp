# Use views::chunk and views::slide for windowed processing (C++23)

**Category:** Ranges (C++20)  
**Item:** #169  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/chunk_view>  

---

## Topic Overview

`views::chunk(n)` and `views::slide(n)` partition a range into sub-ranges of size `n`, but differ in whether windows **overlap**.

### chunk vs slide Visual Comparison

```cpp

Source:  [1] [2] [3] [4] [5] [6] [7] [8] [9]

chunk(3) — non-overlapping:
  [1,2,3]  [4,5,6]  [7,8,9]

slide(3) — overlapping (sliding window):
  [1,2,3]  [2,3,4]  [3,4,5]  [4,5,6]  [5,6,7]  [6,7,8]  [7,8,9]

chunk(3) on 10 elements — last chunk may be smaller:
  [1,2,3]  [4,5,6]  [7,8,9]  [10]

slide(3) on 10 elements — always exactly size 3:
  [1,2,3]  [2,3,4]  ...  [8,9,10]    (8 windows)

```

### Summary Table

| Property | `chunk(n)` | `slide(n)` |
| --- | --- | --- |
| Overlap | None | n-1 elements shared |
| Window count | ⌈size/n⌉ | size - n + 1 |
| Last window size | May be < n | Always n |
| Required | `input_range` | `forward_range` |
| Result element | Sub-range / take_view | Sub-range of original |

### Related: `chunk_by` (C++23)

`views::chunk_by(pred)` splits whenever `pred(prev, curr)` returns `false`—for grouping by a condition rather than fixed size.

---

## Self-Assessment

### Q1: Split a range into fixed-size chunks using `views::chunk` and process each chunk

```cpp

#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Split into chunks of 3
    std::cout << "Chunks of 3:\n";
    for (auto chunk : data | std::views::chunk(3)) {
        std::cout << "  [";
        bool first = true;
        for (int x : chunk) {
            if (!first) std::cout << ", ";
            std::cout << x;
            first = false;
        }
        std::cout << "]\n";
    }

    // Process each chunk: compute the sum of each chunk
    std::cout << "Chunk sums: ";
    for (auto chunk : data | std::views::chunk(4)) {
        int sum = 0;
        for (int x : chunk) sum += x;
        std::cout << sum << ' ';
    }
    std::cout << '\n';

    // Batch processing: simulate sending data in packets of 3
    int batch = 1;
    for (auto packet : data | std::views::chunk(3)) {
        std::cout << "Batch " << batch++ << ": ";
        for (int x : packet) std::cout << x << ' ';
        std::cout << '\n';
    }
}
// Expected output:
// Chunks of 3:
//   [1, 2, 3]
//   [4, 5, 6]
//   [7, 8, 9]
//   [10]
// Chunk sums: 10 26 19
// Batch 1: 1 2 3
// Batch 2: 4 5 6
// Batch 3: 7 8 9
// Batch 4: 10

```

**How this works:**

- `views::chunk(3)` yields sub-ranges of at most 3 elements each.
- The last chunk contains only 1 element (10) since 10 is not divisible by 3.
- Each chunk is itself a range that can be iterated or passed to algorithms.
- Common use: batch processing, pagination, or splitting data for parallel processing.

### Q2: Use `views::slide` to compute a moving average over a range

```cpp

#include <iostream>
#include <numeric>
#include <ranges>
#include <vector>

int main() {
    std::vector<double> prices = {100.0, 102.5, 101.0, 105.0, 103.5, 107.0, 106.0};

    constexpr int window = 3;

    // Compute 3-day moving average
    std::cout << "Prices:          ";
    for (double p : prices) std::cout << p << ' ';
    std::cout << '\n';

    std::cout << "Moving avg (" << window << "): ";
    for (auto win : prices | std::views::slide(window)) {
        double sum = 0.0;
        int count = 0;
        for (double x : win) { sum += x; ++count; }
        std::cout << (sum / count) << ' ';
    }
    std::cout << '\n';

    // Detect consecutive increases using slide(2)
    std::cout << "Consecutive increases: ";
    for (auto pair : prices | std::views::slide(2)) {
        auto it = std::ranges::begin(pair);
        double a = *it++;
        double b = *it;
        if (b > a) std::cout << a << "->" << b << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// Prices:          100 102.5 101 105 103.5 107 106
// Moving avg (3): 101.167 102.833 103.167 105.167 105.5
// Consecutive increases: 100->102.5 101->105 103.5->107

```

**How this works:**

- `views::slide(3)` produces 5 overlapping windows of exactly 3 elements each.
- Each window is a sub-range into the original vector (no copies).
- The moving average divides the sum of each window by the window size.
- `slide(2)` gives adjacent pairs—useful for detecting trends, computing differences, etc.

### Q3: Explain the difference between chunk (non-overlapping) and slide (overlapping windows)

| Aspect | `chunk(n)` | `slide(n)` |
| --- | --- | --- |
| **Windows** | Non-overlapping partitions | Overlapping with stride 1 |
| **Element sharing** | Each element in exactly 1 chunk | Each element in up to `n` windows |
| **Window count** (for `N` elements) | ⌈N/n⌉ | N - n + 1 |
| **Last window** | May have fewer than `n` elements | Always exactly `n` elements |
| **Use case** | Batch processing, pagination | Moving average, trend detection |
| **Minimum requirement** | `input_range` | `forward_range` |

**Demonstration:**

```cpp

#include <iostream>
#include <ranges>
#include <vector>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6};

    std::cout << "chunk(3) — non-overlapping:\n";
    for (auto c : data | std::views::chunk(3)) {
        std::cout << "  ";
        for (int x : c) std::cout << x << ' ';
        std::cout << '\n';
    }

    std::cout << "slide(3) — overlapping:\n";
    for (auto w : data | std::views::slide(3)) {
        std::cout << "  ";
        for (int x : w) std::cout << x << ' ';
        std::cout << '\n';
    }

    // chunk_by: split on condition changes
    std::vector<int> vals = {1, 1, 2, 2, 2, 3, 1, 1};
    std::cout << "chunk_by(equal):\n";
    for (auto group : vals | std::views::chunk_by(std::equal_to{})) {
        std::cout << "  ";
        for (int x : group) std::cout << x << ' ';
        std::cout << '\n';
    }
}
// Expected output:
// chunk(3) — non-overlapping:
//   1 2 3
//   4 5 6
// slide(3) — overlapping:
//   1 2 3
//   2 3 4
//   3 4 5
//   4 5 6
// chunk_by(equal):
//   1 1
//   2 2 2
//   3
//   1 1

```

**Key insight:** `chunk` partitions the data (each element belongs to exactly one chunk), while `slide` creates overlapping windows (elements near the middle appear in `n` different windows). `chunk_by` is condition-based rather than fixed-size.

---

## Notes

- `slide` requires `forward_range` because it needs to iterate the same elements in multiple windows. `chunk` works with `input_range` since each element is visited once.
- `chunk(1)` wraps each element in a single-element sub-range. `slide(1)` does the same.
- `chunk(n)` where `n >= size` produces a single chunk containing all elements.
- Both are **lazy**—sub-ranges are views into the original data.
- Also see `views::stride(n)` for taking every nth element (complementary to chunk).
