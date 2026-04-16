# Implement a custom container satisfying std::ranges::range

**Category:** Standard Library Containers  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/range>  

---

## Topic Overview

To work with `std::ranges` algorithms and views, a container must satisfy the `range` concept: it needs `begin()` and `end()` returning iterators and sentinels.

### Minimal Range-Compatible Container

```cpp

#include <ranges>
#include <iterator>
#include <cstddef>
#include <iostream>
#include <algorithm>

template<typename T, size_t N>
class StaticVec {
    T data_[N]{};
    size_t size_ = 0;

public:
    // Iterator type — a simple pointer
    using iterator = T*;
    using const_iterator = const T*;

    void push_back(const T& val) { data_[size_++] = val; }

    iterator begin() { return data_; }
    iterator end()   { return data_ + size_; }
    const_iterator begin() const { return data_; }
    const_iterator end()   const { return data_ + size_; }

    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
};

// Verify it satisfies the concepts:
static_assert(std::ranges::range<StaticVec<int, 10>>);
static_assert(std::ranges::sized_range<StaticVec<int, 10>>);
static_assert(std::ranges::contiguous_range<StaticVec<int, 10>>);

int main() {
    StaticVec<int, 100> v;
    for (int i : {5, 3, 1, 4, 2}) v.push_back(i);

    // Works with all ranges algorithms:
    std::ranges::sort(v);
    for (int x : v | std::views::filter([](int n) { return n > 2; }))
        std::cout << x << " ";  // 3 4 5
}

```

### Custom Iterator Class

```cpp

#include <iterator>
#include <ranges>

template<typename T>
class ListView {
    struct Node { T value; Node* next; };
    Node* head_ = nullptr;

public:
    class iterator {
        Node* node_;
    public:
        using value_type = T;
        using difference_type = std::ptrdiff_t;
        using iterator_category = std::forward_iterator_tag;

        iterator(Node* n = nullptr) : node_(n) {}
        T& operator*() const { return node_->value; }
        iterator& operator++() { node_ = node_->next; return *this; }
        iterator operator++(int) { auto tmp = *this; ++*this; return tmp; }
        bool operator==(const iterator& o) const { return node_ == o.node_; }
    };

    iterator begin() { return iterator{head_}; }
    iterator end()   { return iterator{nullptr}; }
};

static_assert(std::input_iterator<ListView<int>::iterator>);
static_assert(std::ranges::range<ListView<int>>);

```

---

## Self-Assessment

### Q1: What are the minimum requirements for `std::ranges::range`

A type `R` must have `ranges::begin(r)` and `ranges::end(r)` that return an iterator and a sentinel. The iterator must satisfy `std::input_or_output_iterator`. That's it — no `size()` needed for basic range.

### Q2: What additional concepts enable more algorithms

`sized_range` (enables `ranges::size`), `forward_range` (multi-pass iteration), `bidirectional_range` (reverse iteration), `random_access_range` (subscript, sort), `contiguous_range` (pointer arithmetic, `ranges::data`).

### Q3: What are sentinel types and why do they matter

A sentinel marks the end of a range without being an iterator. It can be a different type from the iterator. This allows lazy ranges (end condition checked per element), null-terminated strings (`sentinel == '\0'`), and infinite ranges.

---

## Notes

- Pointers are valid contiguous iterators — `T*` satisfies `std::contiguous_iterator`.
- Use `static_assert` to verify your container satisfies expected concepts.
- `std::ranges::to<Container>()` (C++23) materializes any range into a container.
- Sentinel/iterator type mismatch is the most common issue when writing custom ranges.
