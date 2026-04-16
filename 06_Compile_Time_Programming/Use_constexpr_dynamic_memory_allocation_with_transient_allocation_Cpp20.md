# Use constexpr dynamic memory allocation with transient allocation (C++20)

**Category:** Compile-Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constexpr>  

---

## Topic Overview

C++20 allows `new` and `delete` inside `constexpr` functions, but all allocations must be freed before the constant expression finishes. This is called "transient allocation."

### Basic Example

```cpp

#include <algorithm>
#include <memory>

constexpr int compute_sum() {
    int* arr = new int[5]{1, 2, 3, 4, 5};
    int sum = 0;
    for (int i = 0; i < 5; ++i) sum += arr[i];
    delete[] arr;  // Must free before returning
    return sum;
}

constexpr int result = compute_sum();  // OK: 15
// All allocations are freed — "transient"

```

### constexpr std::vector and std::string

```cpp

#include <vector>
#include <string>
#include <algorithm>
#include <numeric>

constexpr int sum_of_sorted() {
    std::vector<int> v = {5, 3, 1, 4, 2};
    std::sort(v.begin(), v.end());
    return std::accumulate(v.begin(), v.end(), 0);
    // v's internal buffer is freed when v is destroyed
    // → transient allocation → OK
}

constexpr int answer = sum_of_sorted();  // 15

constexpr size_t count_words() {
    std::string s = "hello world foo bar";
    size_t count = 1;
    for (char c : s)
        if (c == ' ') ++count;
    return count;
    // s freed at end → transient
}

constexpr size_t words = count_words();  // 4

```

### Non-Transient Allocation (Compile Error)

```cpp

constexpr auto make_vector() {
    return std::vector{1, 2, 3};  // Vector escapes constexpr context
}

// constexpr auto v = make_vector();
// ERROR: allocation is NOT transient — it "leaks" into runtime
// The vector's heap memory can't become a compile-time constant

// This works IF the result is used only at compile time:
constexpr auto sz = make_vector().size();  // OK: vector created and destroyed

```

---

## Self-Assessment

### Q1: What makes an allocation "transient"

A transient allocation is one that is both allocated AND deallocated during the same constant expression evaluation. If the allocation "escapes" (the pointer is returned or stored), it's non-transient and causes a compile error.

### Q2: Can you have constexpr std::map

Yes in C++20 (if your standard library implements it as constexpr). The requirement is that all allocations are transient. You can populate a map, query it, and get a scalar result — all at compile time. You cannot return the map.

### Q3: What is the practical use case

Compile-time computation of lookup tables, string parsing, configuration validation, hash computation, sorting, and any algorithm that needs dynamic memory internally but produces a fixed-size result.

---

## Notes

- `constexpr std::vector/string` work in GCC 12+, Clang 15+, MSVC 19.29+.
- All standard library algorithms are `constexpr` since C++20.
- Non-transient allocations are proposed for C++26 (P2670 — static storage from constexpr).
- This enables "compile-time programming with runtime data structures."
