# Provide noexcept guarantees on move operations for container compatibility

**Category:** API & Library Design  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/types/is_nothrow_move_constructible>  

---

## Topic Overview

`std::vector` and other containers use `std::move_if_noexcept` during reallocation. If your type's move constructor is not `noexcept`, vector falls back to copying — potentially a massive performance penalty.

### The Problem

```cpp

#include <vector>
#include <iostream>
#include <string>

struct Slow {
    std::string data;
    // Move constructor NOT noexcept
    Slow(Slow&& other) : data(std::move(other.data)) {
        // Missing noexcept!
    }
};

struct Fast {
    std::string data;
    // Move constructor IS noexcept
    Fast(Fast&& other) noexcept : data(std::move(other.data)) {}
};

int main() {
    // Slow: vector COPIES during reallocation (for strong exception guarantee)
    std::vector<Slow> v1;
    for (int i = 0; i < 10000; ++i) v1.push_back(Slow{"data"});
    // Many unnecessary copies!

    // Fast: vector MOVES during reallocation
    std::vector<Fast> v2;
    for (int i = 0; i < 10000; ++i) v2.push_back(Fast{"data"});
    // Efficient moves!

    std::cout << "Slow is nothrow move: "
              << std::is_nothrow_move_constructible_v<Slow> << "\n";  // 0
    std::cout << "Fast is nothrow move: "
              << std::is_nothrow_move_constructible_v<Fast> << "\n";  // 1
}

```

### Conditional noexcept

```cpp

#include <type_traits>

template<typename T>
class Wrapper {
    T value_;
public:
    // Propagate noexcept from T's move constructor
    Wrapper(Wrapper&& other) noexcept(std::is_nothrow_move_constructible_v<T>)
        : value_(std::move(other.value_)) {}

    Wrapper& operator=(Wrapper&& other)
        noexcept(std::is_nothrow_move_assignable_v<T>) {
        value_ = std::move(other.value_);
        return *this;
    }
};

```

---

## Self-Assessment

### Q1: How does std::move_if_noexcept work

```cpp

// Simplified implementation:
template<typename T>
auto move_if_noexcept(T& x) noexcept {
    if constexpr (std::is_nothrow_move_constructible_v<T>)
        return std::move(x);  // Move — safe, no exceptions
    else
        return x;             // Copy — safe, strong guarantee
}

```

### Q2: Show the performance impact with a benchmark

```cpp

// With noexcept move: vector reallocation is O(n) moves ≈ O(n) pointer swaps
// Without noexcept move: vector reallocation is O(n) copies ≈ O(n) string copies
// For std::string of 1KB: moves are ~100x faster than copies
// For a vector growing to 10000 elements: total difference is ~10ms vs ~1s

```

### Q3: What about swap

```cpp

class MyType {
    std::vector<int> data_;
public:
    // Swap should ALWAYS be noexcept
    friend void swap(MyType& a, MyType& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);  // vector::swap is noexcept
    }

    // Copy-and-swap assignment leverages noexcept swap
    MyType& operator=(MyType other) noexcept {
        swap(*this, other);
        return *this;
    }
};

```

---

## Notes

- **Rule**: move constructors and assignment operators should always be `noexcept` if possible.
- Use `static_assert(std::is_nothrow_move_constructible_v<MyType>)` to verify.
- If your type wraps only standard library types, defaulted move operations are already `noexcept`.
- Use `noexcept(noexcept(...))` for conditional propagation in generic code.
