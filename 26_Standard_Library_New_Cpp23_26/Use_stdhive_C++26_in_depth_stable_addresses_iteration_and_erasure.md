# Use std::hive (C++26) in depth: stable addresses, iteration, and erasure

**Category:** Standard Library — New in C++23/26  
**Item:** #581  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/container/hive>  

---

## Topic Overview

`std::hive` (C++26) is a **bucket-based container** optimized for workloads with frequent **random insertion and erasure** — common in game engines (entity pools), simulations (particle systems), and GUI frameworks (widget pools).

### Key Properties

| Property                | Description                                                 |
| --- | --- |
| **Stable addresses**    | Pointers/references to elements remain valid after insert/erase |
| **Automatic hole skipping** | Iteration skips erased slots without user intervention  |
| **No compaction needed** | Unlike vector: erase leaves a "hole", no element shifting  |
| **Cache-friendly**      | Elements stored in contiguous blocks (chunks)              |
| **Unordered**           | No guaranteed element ordering; new inserts may fill holes  |

### How It Works

```cpp

Block 0: [A][B][ ][D][E]     ← slot 2 erased, becomes a "hole"
Block 1: [F][ ][ ][I][J]     ← slots 1,2 are holes
                  ↓
Iteration: A → B → D → E → F → I → J   (holes skipped automatically)
Insert: new element fills an existing hole (O(1))

```

---

## Self-Assessment

### Q1: Insert and erase elements in a std::hive and verify that surviving element addresses remain stable

**Answer:**

```cpp

// C++26 — requires <hive> support
#include <hive>
#include <iostream>
#include <vector>
#include <cassert>

int main() {
    std::hive<int> h;

    // Insert elements and record their addresses
    auto it1 = h.insert(10);
    auto it2 = h.insert(20);
    auto it3 = h.insert(30);

    int* addr1 = &*it1;  // Address of element 10
    int* addr3 = &*it3;  // Address of element 30

    std::cout << "Before erase:\n";
    std::cout << "  *addr1 = " << *addr1 << " (at " << addr1 << ")\n";
    std::cout << "  *addr3 = " << *addr3 << " (at " << addr3 << ")\n";

    // Erase the middle element
    h.erase(it2);  // Removes 20, creates a hole

    // KEY GUARANTEE: addresses of surviving elements are UNCHANGED
    std::cout << "After erasing 20:\n";
    std::cout << "  *addr1 = " << *addr1 << " (at " << addr1 << ")\n";  // Still 10
    std::cout << "  *addr3 = " << *addr3 << " (at " << addr3 << ")\n";  // Still 30
    assert(*addr1 == 10);  // ✓ Stable!
    assert(*addr3 == 30);  // ✓ Stable!

    // Insert new element — may reuse the hole left by 20
    auto it4 = h.insert(40);
    assert(*addr1 == 10);  // Still stable after insert
    assert(*addr3 == 30);

    // Print all elements (holes are skipped)
    std::cout << "All elements: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // Possible output: 10 40 30  (40 may fill the hole where 20 was)

    // Contrast with std::vector:
    // std::vector<int> v = {10, 20, 30};
    // int* vaddr = &v[0];
    // v.erase(v.begin() + 1);
    // *vaddr may now be 20 (shifted!) — NOT STABLE
}

```

### Q2: Show that std::hive iteration skips holes (erased slots) automatically without repack

**Answer:**

```cpp

#include <hive>
#include <iostream>

int main() {
    std::hive<int> h;

    // Insert 10 elements: 0, 1, 2, ..., 9
    std::vector<std::hive<int>::iterator> iters;
    for (int i = 0; i < 10; ++i)
        iters.push_back(h.insert(i));

    std::cout << "Before erasure (" << h.size() << " elements): ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 0 1 2 3 4 5 6 7 8 9

    // Erase every other element → creates holes at indices 1, 3, 5, 7, 9
    for (int i = 1; i < 10; i += 2)
        h.erase(iters[i]);

    std::cout << "After erasing odd indices (" << h.size() << " elements): ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 0 2 4 6 8  — holes skipped automatically!

    // No repack/compaction happened — elements 0,2,4,6,8 are at their
    // ORIGINAL memory addresses. The iteration logic uses a skipfield
    // to jump over erased slots efficiently.

    // Internal structure (conceptual):
    // Memory:    [0][ ][2][ ][4][ ][6][ ][8][ ]
    // Skipfield: [0][1][0][1][0][1][0][1][0][1]
    //               ^     ^     ^     ^     ^  = holes (skipped)

    // Erasing more elements just adds more holes
    h.erase(h.begin());  // Erase element 0
    std::cout << "After erasing first: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 2 4 6 8

    // Inserting fills holes first (no allocation if holes exist)
    h.insert(99);
    std::cout << "After inserting 99: ";
    for (int x : h) std::cout << x << ' ';
    std::cout << '\n';
    // 99 2 4 6 8  (99 may occupy a hole)
}

```

### Q3: Benchmark std::hive vs std::list and std::deque for a workload with frequent random erasure

**Answer:**

```cpp

#include <hive>
#include <list>
#include <deque>
#include <vector>
#include <chrono>
#include <iostream>
#include <random>
#include <algorithm>

template<typename Container>
double benchmark_insert_erase(int n_elements, int n_erasures) {
    Container c;
    std::vector<typename Container::iterator> iters;
    iters.reserve(n_elements);

    // Phase 1: Insert N elements
    for (int i = 0; i < n_elements; ++i)
        iters.push_back(c.insert(c.end(), i));

    // Phase 2: Randomly erase half the elements
    std::mt19937 rng(42);
    std::shuffle(iters.begin(), iters.end(), rng);

    auto start = std::chrono::high_resolution_clock::now();

    for (int i = 0; i < n_erasures && !iters.empty(); ++i) {
        c.erase(iters.back());
        iters.pop_back();
    }

    // Phase 3: Iterate over surviving elements (sum)
    volatile int sum = 0;
    for (auto& x : c) sum += x;

    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(end - start).count();
}

int main() {
    constexpr int N = 100'000;
    constexpr int E = N / 2;

    // Note: std::hive<int> requires C++26 compiler support
    // Here we show the expected performance characteristics
    double hive_ms = benchmark_insert_erase<std::hive<int>>(N, E);
    double list_ms = benchmark_insert_erase<std::list<int>>(N, E);
    // std::deque doesn't support stable erase-by-iterator in the same way
    // Included for reference with sequential erasure

    std::cout << "Erase " << E << " of " << N << " + iterate survivors:\n";
    std::cout << "  std::hive: " << hive_ms << " ms\n";
    std::cout << "  std::list: " << list_ms << " ms\n";

    /*
    Expected results (100K elements, erase 50K):
    ┌──────────────┬───────────┬───────────────────────────────┐
    │ Container    │ Time      │ Why                           │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::hive    │ ~2-4 ms   │ O(1) erase + cache-friendly   │
    │              │           │ iteration with skipfield       │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::list    │ ~8-15 ms  │ O(1) erase but pointer-chasing│
    │              │           │ on iteration (cache misses)    │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::vector  │ ~50+ ms   │ O(n) erase (shifts elements)  │
    │              │           │ but fast iteration             │
    ├──────────────┼───────────┼───────────────────────────────┤
    │ std::deque   │ ~30+ ms   │ O(n) erase for random pos     │
    │              │           │ moderate iteration speed       │
    └──────────────┴───────────┴───────────────────────────────┘

    std::hive wins when:

    - Frequent random erasure (entities dying, particles expiring)
    - External pointers to elements must remain valid
    - Iteration over survivors is a hot-path operation

    */
}

```

---

## Notes

- `std::hive` was previously known as `plf::colony` (by Matthew Bentley) — battle-tested in game engines
- Elements are stored in blocks; erased slots become holes tracked by a **skipfield** (O(1) skip per hole during iteration)
- `hive::reshape()` controls the min/max block sizes for tuning memory overhead vs fragmentation
- Not random-access: no `operator[]`. Use iterators only
- Ideal for **entity-component-system (ECS)** pools where entities are created/destroyed frequently
