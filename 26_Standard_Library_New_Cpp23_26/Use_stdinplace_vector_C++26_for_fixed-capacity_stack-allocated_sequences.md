# Use std::inplace_vector (C++26) for fixed-capacity stack-allocated sequences

**Category:** Standard Library — New in C++23/26  
**Item:** #580  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/inplace_vector>  

---

## Topic Overview

`std::inplace_vector<T, N>` (C++26, `<inplace_vector>`) is a **fixed-capacity, variable-size container** that stores up to `N` elements entirely on the stack. It provides the same interface as `std::vector` but **never heap-allocates**.

### Key Properties

| Property              | `std::vector<T>`          | `std::inplace_vector<T,N>` | `std::array<T,N>`       |
| --- | --- | --- | --- |
| **Storage**           | Heap                      | Stack (inline)              | Stack (inline)          |
| **Size**              | Dynamic                   | Variable (0..N)             | Fixed (always N)        |
| **push_back**         | O(1) amortized            | O(1) or throws              | N/A                     |
| **Heap allocation**   | Yes                       | Never                       | Never                   |
| **constexpr**         | Partial (C++20)           | Full                        | Full                    |
| **Contiguous**        | Yes                       | Yes                         | Yes                     |

### When to Use

- **Embedded / real-time** — no allocator, deterministic timing
- **Hot paths** — avoid allocator overhead when max size is known and small
- **constexpr** — compute arrays at compile time with dynamic logic

---

## Self-Assessment

### Q1: Replace a std::vector in a hot path with inplace_vector<T,N> and verify zero heap allocation

**Answer:**

```cpp

#include <inplace_vector>  // C++26
#include <vector>
#include <iostream>
#include <chrono>
#include <cstdlib>

// Scenario: collecting up to 8 neighbors in a grid computation (hot path)

struct Point { int x, y; };

// OLD: heap allocation per call
std::vector<Point> get_neighbors_heap(int x, int y) {
    std::vector<Point> result;
    result.reserve(8);  // Still allocates 8 * sizeof(Point) on heap
    for (int dx = -1; dx <= 1; ++dx)
        for (int dy = -1; dy <= 1; ++dy)
            if (dx != 0 || dy != 0)
                result.push_back({x + dx, y + dy});
    return result;
}

// NEW: zero heap allocation
std::inplace_vector<Point, 8> get_neighbors_stack(int x, int y) {
    std::inplace_vector<Point, 8> result;
    for (int dx = -1; dx <= 1; ++dx)
        for (int dy = -1; dy <= 1; ++dy)
            if (dx != 0 || dy != 0)
                result.push_back({x + dx, y + dy});
    return result;  // sizeof(result) == sizeof(Point)*8 + size_t ≈ 72 bytes on stack
}

int main() {
    constexpr int ITERS = 1'000'000;

    auto t1 = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERS; ++i) {
        auto n = get_neighbors_heap(50, 50);
        volatile auto s = n.size();
    }
    auto t2 = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < ITERS; ++i) {
        auto n = get_neighbors_stack(50, 50);
        volatile auto s = n.size();
    }
    auto t3 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "std::vector (heap):          " << ms(t2 - t1).count() << " ms\n";
    std::cout << "std::inplace_vector (stack): " << ms(t3 - t2).count() << " ms\n";
    // Typical result: inplace_vector 3-5x faster due to zero allocation

    // Verify: inplace_vector content is correct
    auto neighbors = get_neighbors_stack(5, 5);
    std::cout << "Neighbors of (5,5): " << neighbors.size() << " points\n";
    for (auto [x, y] : neighbors)
        std::cout << "  (" << x << "," << y << ")\n";
}

```

### Q2: Show that try_push_back returns false instead of throwing when capacity is reached

**Answer:**

```cpp

#include <inplace_vector>  // C++26
#include <iostream>

int main() {
    std::inplace_vector<int, 4> v;

    // push_back: throws std::bad_alloc when full
    v.push_back(10);
    v.push_back(20);
    v.push_back(30);
    v.push_back(40);
    std::cout << "Size: " << v.size() << "/" << v.capacity() << '\n';  // 4/4

    try {
        v.push_back(50);  // Throws! Cannot exceed capacity 4
    } catch (const std::bad_alloc& e) {
        std::cout << "push_back threw: " << e.what() << '\n';
    }

    // try_push_back: returns pointer (success) or nullptr (full) — NO exception
    std::inplace_vector<int, 4> v2;
    for (int i = 1; i <= 6; ++i) {
        auto* result = v2.try_push_back(i * 10);
        if (result) {
            std::cout << "Inserted " << *result << " (size=" << v2.size() << ")\n";
        } else {
            std::cout << "FULL — cannot insert " << i * 10 << '\n';
        }
    }
    // Output:
    // Inserted 10 (size=1)
    // Inserted 20 (size=2)
    // Inserted 30 (size=3)
    // Inserted 40 (size=4)
    // FULL — cannot insert 50
    // FULL — cannot insert 60

    // try_emplace_back: same pattern for in-place construction
    struct Widget {
        std::string name;
        int value;
    };
    std::inplace_vector<Widget, 2> widgets;
    auto* w1 = widgets.try_emplace_back("alpha", 1);  // OK
    auto* w2 = widgets.try_emplace_back("beta", 2);   // OK
    auto* w3 = widgets.try_emplace_back("gamma", 3);  // nullptr — full!
    std::cout << "w3 is null: " << (w3 == nullptr) << '\n';  // 1

    // unchecked_push_back: no check at all — UB if full (fastest)
    std::inplace_vector<int, 8> fast;
    fast.unchecked_push_back(42);  // Caller guarantees size < capacity
}

```

### Q3: Explain how inplace_vector satisfies ContiguousContainer and works with std::span

**Answer:**

```cpp

#include <inplace_vector>  // C++26
#include <span>
#include <iostream>
#include <algorithm>
#include <numeric>

// inplace_vector<T,N> satisfies ContiguousContainer:
// - data() returns T* to first element
// - Elements are contiguous in memory
// - Compatible with std::span, C APIs, SIMD operations

// Generic function taking a span — works with any contiguous source
double compute_average(std::span<const int> data) {
    if (data.empty()) return 0.0;
    return static_cast<double>(
        std::accumulate(data.begin(), data.end(), 0)) / data.size();
}

void process_buffer(std::span<int> buf) {
    std::sort(buf.begin(), buf.end());
}

// C API interop
extern "C" {
    // void legacy_api(const int* data, int count);
}

int main() {
    std::inplace_vector<int, 8> iv = {5, 3, 8, 1, 9, 2};

    // ═══════════ Pass to span-based functions ═══════════
    double avg = compute_average(iv);  // Implicit conversion to span
    std::cout << "Average: " << avg << '\n';  // 4.666...

    process_buffer(iv);  // Sorts in-place via span
    std::cout << "Sorted: ";
    for (int x : iv) std::cout << x << ' ';
    std::cout << '\n';  // 1 2 3 5 8 9

    // ═══════════ Explicit span construction ═══════════
    std::span<int> s(iv);
    std::span<int> first3 = s.first(3);
    std::cout << "First 3: ";
    for (int x : first3) std::cout << x << ' ';
    std::cout << '\n';  // 1 2 3

    // ═══════════ data() for C API interop ═══════════
    int* raw = iv.data();
    size_t count = iv.size();
    // legacy_api(raw, static_cast<int>(count));  // Pass to C function

    // ═══════════ constexpr usage ═══════════
    constexpr auto make_primes() {
        std::inplace_vector<int, 10> primes;
        for (int n = 2; primes.size() < 10; ++n) {
            bool is_prime = true;
            for (size_t i = 0; i < primes.size(); ++i) {
                if (n % primes[i] == 0) { is_prime = false; break; }
            }
            if (is_prime) primes.push_back(n);
        }
        return primes;
    }
    constexpr auto primes = make_primes();
    static_assert(primes[0] == 2);
    static_assert(primes[9] == 29);
    std::cout << "10th prime (constexpr): " << primes[9] << '\n';  // 29
}

```

---

## Notes

- `std::inplace_vector` is C++26 (`<inplace_vector>`); reference implementation available (Boost.Container has `static_vector`)
- Three insertion modes: `push_back` (throws), `try_push_back` (returns null), `unchecked_push_back` (UB if full, fastest)
- `sizeof(inplace_vector<T,N>)` = `sizeof(T) * N + sizeof(size_type)` — entirely on the stack
- Ideal for small buffers in hot loops: neighboring cells, stack-allocated scratch space, embedded systems
- Fully `constexpr` — can compute sequences at compile time with push_back logic
