# Implement a custom container satisfying std::ranges::range

**Category:** Standard Library Containers  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/range>  

---

## Topic Overview

To work with `std::ranges` algorithms and views, a container must satisfy the `range` concept. The good news is that the bar is actually pretty low: all you need is `begin()` and `end()` that return an iterator and a sentinel the library can work with. Once you clear that bar, your custom type plugs into the entire ranges ecosystem.

### Minimal Range-Compatible Container

Here's the smallest example that checks out. `StaticVec` is a fixed-size container that stores elements inline. Notice the `static_assert` lines at the bottom - those are your free unit tests that run at compile time and tell you exactly which concept you failed if something is missing.

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
    // Iterator type - a simple pointer
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

Because `T*` satisfies `std::contiguous_iterator`, the compiler promotes your raw-pointer iterators all the way up the hierarchy. You get `sized_range` because `size()` is present, and `contiguous_range` because the data is one flat block of memory. Not bad for about fifteen lines of interface code.

### Custom Iterator Class

When your data is not contiguous - say, a linked list - you can't use a raw pointer. Instead you write a small iterator class. The key is to advertise what category of iterator you are via `iterator_category`, and to implement exactly the operations that category requires. Here a forward-linked list only needs `++`, `*`, and `==`:

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

`nullptr` serves as the sentinel here - it represents the past-the-end position. This is totally valid; sentinels do not have to be the same type as the iterator, and they do not have to be a real element address.

---

## Self-Assessment

### Q1: What are the minimum requirements for `std::ranges::range`

A type `R` must have `ranges::begin(r)` and `ranges::end(r)` that return an iterator and a sentinel. The iterator must satisfy `std::input_or_output_iterator`. That's it - no `size()` needed for basic range.

### Q2: What additional concepts enable more algorithms

`sized_range` (enables `ranges::size`), `forward_range` (multi-pass iteration), `bidirectional_range` (reverse iteration), `random_access_range` (subscript, sort), `contiguous_range` (pointer arithmetic, `ranges::data`).

### Q3: What are sentinel types and why do they matter

A sentinel marks the end of a range without being an iterator. It can be a different type from the iterator. This allows lazy ranges (end condition checked per element), null-terminated strings (`sentinel == '\0'`), and infinite ranges.

---

## Notes

- Pointers are valid contiguous iterators - `T*` satisfies `std::contiguous_iterator`.
- Use `static_assert` to verify your container satisfies expected concepts.
- `std::ranges::to<Container>()` (C++23) materializes any range into a container.
- Sentinel/iterator type mismatch is the most common issue when writing custom ranges.
