# Use std::vector correctly: reserve, emplace_back, and iterator invalidation

**Category:** Standard Library — Containers  
**Item:** #61  
**Reference:** <https://en.cppreference.com/w/cpp/container/vector>  

---

## Topic Overview

`std::vector` is the most used C++ container — a dynamic array with contiguous memory. Understanding `reserve`, `emplace_back`, and iterator invalidation is critical for writing correct, performant code.

### Memory Model

```cpp

vector v = {1, 2, 3}

Stack:               Heap:
┌────────────┐       ┌───┬───┬───┬───┬───┐
│ data ──────────────→│ 1 │ 2 │ 3 │   │   │
│ size = 3   │       └───┴───┴───┴───┴───┘
│ capacity = 5│       ←── size ──→
└────────────┘       ←──── capacity ─────→

push_back(4): fits in capacity → just writes to slot 3. Size becomes 4.
push_back(6) after 5: exceeds capacity → allocate new larger buffer, copy all elements, free old buffer.

```

### Key Operations

| Operation | Effect | Complexity |
| --- | --- | --- |
| `push_back(x)` | Copy/move x into vector | Amortized O(1) |
| `emplace_back(args...)` | Construct in-place | Amortized O(1) |
| `reserve(n)` | Ensure capacity ≥ n (no size change) | O(n) if reallocation |
| `resize(n)` | Change size to n (default-constructs new elements) | O(n) |
| `shrink_to_fit()` | Request reduce capacity to size | Implementation-defined |
| `insert(pos, x)` | Insert at position | O(n) — shifts elements |
| `erase(pos)` | Remove at position | O(n) — shifts elements |

### Iterator Invalidation Rules

| Operation | Invalidated iterators |
| --- | --- |
| `push_back` / `emplace_back` | All, **if** reallocation occurs. Otherwise none. |
| `insert` | All if reallocation. Past-the-insertion-point otherwise. |
| `erase` | Past-the-erasure-point and `end()` |
| `reserve` | All, **if** reallocation occurs |
| `resize` (growing) | All if reallocation |
| `clear` | All |
| `swap` | All (they refer to swapped-from vector) |
| Read-only (`operator[]`, `at`, iteration) | None |

---

## Self-Assessment

### Q1: Explain when iterators are invalidated in a vector and write a bug that demonstrates this

```cpp

#include <iostream>
#include <vector>

int main() {
    // === BUG: iterator invalidation during push_back ===
    std::vector<int> v = {1, 2, 3, 4, 5};

    std::cout << "Before: size=" << v.size() << " capacity=" << v.capacity() << "\n";

    // Save an iterator to the third element
    auto it = v.begin() + 2;
    std::cout << "Iterator points to: " << *it << "\n";  // 3

    // Force reallocation by exceeding capacity
    while (v.size() < v.capacity()) v.push_back(99);  // fill to capacity
    std::cout << "Filled to capacity: size=" << v.size() << "\n";

    // This push_back WILL reallocate
    v.push_back(100);
    std::cout << "After reallocation: size=" << v.size()
              << " capacity=" << v.capacity() << "\n";

    // ⚠️ UNDEFINED BEHAVIOR: `it` now points to freed memory!
    // std::cout << "*it = " << *it << "\n";  // CRASH or garbage

    // === BUG: erasing during iteration ===
    std::vector<int> nums = {1, 2, 3, 4, 5, 6};

    // WRONG: erase invalidates iterators past the erasure point
    // for (auto it = nums.begin(); it != nums.end(); ++it) {
    //     if (*it % 2 == 0) nums.erase(it);  // UNDEFINED BEHAVIOR: it is now invalid
    // }

    // CORRECT: erase returns the next valid iterator
    for (auto it = nums.begin(); it != nums.end(); ) {
        if (*it % 2 == 0)
            it = nums.erase(it);   // erase returns iterator to next element
        else
            ++it;
    }

    std::cout << "After removing evens: ";
    for (int n : nums) std::cout << n << " ";
    std::cout << "\n";  // 1 3 5

    // EVEN BETTER (C++20):
    std::vector<int> nums2 = {1, 2, 3, 4, 5, 6};
    std::erase_if(nums2, [](int x) { return x % 2 == 0; });
    std::cout << "C++20 erase_if:     ";
    for (int n : nums2) std::cout << n << " ";
    std::cout << "\n";  // 1 3 5

    return 0;
}

```

**How it works:**

- **Reallocation invalidation:** When `push_back` exceeds capacity, vector allocates a new buffer, copies/moves elements, and frees the old buffer. Any iterator, pointer, or reference to the old buffer is now **dangling**.
- **Erase invalidation:** `erase(pos)` shifts all elements after `pos` forward. The iterator at `pos` now points to the next element (or `end()`). Iterators past `pos` are invalidated.
- **Safe patterns:** Use index-based loops instead of iterators when modifying the vector, or use the iterator returned by `erase()`. C++20's `std::erase`/`std::erase_if` handles this correctly.

### Q2: Show that emplace_back avoids an extra move constructor call compared to push_back with a temporary

```cpp

#include <iostream>
#include <vector>
#include <string>

struct Widget {
    std::string name;
    int value;

    Widget(std::string n, int v) : name(std::move(n)), value(v) {
        std::cout << "  Constructor: Widget(" << name << ", " << value << ")\n";
    }
    Widget(const Widget& other) : name(other.name), value(other.value) {
        std::cout << "  Copy ctor: Widget(" << name << ")\n";
    }
    Widget(Widget&& other) noexcept : name(std::move(other.name)), value(other.value) {
        std::cout << "  Move ctor: Widget(" << name << ")\n";
    }
};

int main() {
    std::vector<Widget> v;
    v.reserve(4);  // prevent reallocation to isolate the effect

    // === push_back with temporary ===
    std::cout << "push_back(Widget{\"A\", 1}):\n";
    v.push_back(Widget{"A", 1});
    // Output:
    //   Constructor: Widget(A, 1)     ← temporary constructed
    //   Move ctor: Widget(A)          ← moved into vector storage

    // === emplace_back with constructor args ===
    std::cout << "\nemplace_back(\"B\", 2):\n";
    v.emplace_back("B", 2);
    // Output:
    //   Constructor: Widget(B, 2)     ← constructed directly in vector storage
    // No move/copy! One fewer constructor call.

    // === push_back with existing object ===
    std::cout << "\npush_back(existing):\n";
    Widget w{"C", 3};
    v.push_back(std::move(w));
    // Output:
    //   Constructor: Widget(C, 3)     ← w constructed
    //   Move ctor: Widget(C)          ← moved into vector

    // === Summary ===
    std::cout << "\nAll widgets:\n";
    for (const auto& w : v)
        std::cout << "  " << w.name << " = " << w.value << "\n";
    // A = 1
    // B = 2
    // C = 3

    // === When does it matter? ===
    // emplace_back saves 1 move when constructing from args.
    // For cheap-to-move types (string, vector), the difference is tiny.
    // For expensive types (no move ctor, large objects), it matters more.
    // emplace_back can also construct from implicit conversion:
    std::vector<std::string> sv;
    sv.reserve(2);
    sv.push_back("hello");      // constructs temp string, then moves
    sv.emplace_back("world");   // constructs string directly in-place

    return 0;
}

```

**How it works:**

- `push_back(Widget{"A", 1})`: Creates a temporary Widget, then **move-constructs** it into the vector. Two constructor calls total.
- `emplace_back("B", 2)`: Forwards the arguments directly to Widget's constructor, constructing the object **in-place** inside the vector's storage. One constructor call total.
- The saving is exactly one move (or copy) constructor call per insertion.
- For types with expensive copy/move (e.g., large buffers), `emplace_back` can be significantly faster.

### Q3: Demonstrate using reserve before a loop to avoid repeated reallocations

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <string>
#include <cstdlib>

// Track allocations
static int alloc_count = 0;
void* operator new(std::size_t sz) {
    alloc_count++;
    return std::malloc(sz);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

int main() {
    constexpr int N = 100'000;

    // === WITHOUT reserve ===
    alloc_count = 0;
    {
        std::vector<int> v;
        for (int i = 0; i < N; ++i)
            v.push_back(i);
        std::cout << "Without reserve: " << alloc_count << " allocations\n";
        std::cout << "  Final capacity: " << v.capacity() << "\n";
    }
    // Typical output: ~17 allocations (capacity doubles: 1,2,4,8,16,...,131072)
    // Each reallocation copies ALL existing elements!

    // === WITH reserve ===
    alloc_count = 0;
    {
        std::vector<int> v;
        v.reserve(N);  // ONE allocation for the full size
        for (int i = 0; i < N; ++i)
            v.push_back(i);
        std::cout << "\nWith reserve:    " << alloc_count << " allocations\n";
        std::cout << "  Final capacity: " << v.capacity() << "\n";
    }
    // Output: 1 allocation (exactly what we needed)

    // === Performance comparison ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<int> v;
        for (int i = 0; i < N; ++i) v.push_back(i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "\nWithout reserve: " << us << " us\n";
    }
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<int> v;
        v.reserve(N);
        for (int i = 0; i < N; ++i) v.push_back(i);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "With reserve:    " << us << " us\n";
    }

    // === reserve vs resize ===
    std::vector<int> r1, r2;
    r1.reserve(10);   // capacity=10, size=0  (no elements created)
    r2.resize(10);    // capacity=10, size=10 (10 zero-initialized ints)

    std::cout << "\nreserve(10): size=" << r1.size() << " cap=" << r1.capacity() << "\n";
    std::cout << "resize(10):  size=" << r2.size() << " cap=" << r2.capacity() << "\n";

    // reserve: use push_back/emplace_back to add elements
    // resize:  elements already exist, access with operator[]

    return 0;
}

```

**How it works:**

- **Without reserve:** Vector starts with capacity 0(or 1). Each time `push_back` exceeds capacity, a new buffer (typically 2× larger) is allocated, all elements are copied/moved, and the old buffer is freed. For N elements, this causes O(log N) reallocations and O(N log N) total copies.
- **With reserve:** A single allocation for the exact needed capacity. No reallocations, no copies. O(N) total.
- **`reserve` vs `resize`:** `reserve` only allocates memory (size stays 0). `resize` also default-constructs elements (size becomes n). Use `reserve` when filling with `push_back`; use `resize` when you need immediate indexed access.
- **When to use:** Always call `reserve` when you know (or can estimate) the number of elements before a loop.

---

## Notes

- **Growth factor:** Most implementations use 2× (GCC, Clang) or 1.5× (MSVC). Neither is mandated by the standard.
- **`shrink_to_fit()`** is a non-binding request — the implementation may ignore it.
- **`emplace_back` pitfall:** `v.emplace_back(v[0])` can be UB if the emplace triggers reallocation — the reference to `v[0]` dangles after reallocation. Use `push_back` or copy first.
- **Prefer `emplace_back`** when constructing from multiple arguments. For single values, `push_back` is equally efficient (and clearer).
- **C++20 `std::erase` / `std::erase_if`:** Replace the erase-remove idiom. `std::erase(v, 3)` removes all elements equal to 3.
- **Contiguous memory guarantee:** `&v[0]` is a valid pointer to a C-style array of `v.size()` elements. You can pass it to C APIs.
