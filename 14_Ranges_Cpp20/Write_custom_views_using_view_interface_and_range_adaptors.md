# Write custom views using view_interface and range adaptors

**Category:** Ranges (C++20)  
**Item:** #119  
**Standard:** C++20 / C++23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/view_interface>  

---

## Topic Overview

`std::ranges::view_interface<Derived>` is a CRTP base class that provides default implementations for common range operations based on `begin()` and `end()` you provide.

### What view_interface Gives You for Free

| Method | Requires from Derived | What it provides |
| --- | --- | --- |
| `empty()` | `begin()`, `end()` | `begin() == end()` |
| `operator bool` | `empty()` | `!empty()` |
| `front()` | forward `begin()` | `*begin()` |
| `back()` | bidirectional + common | `*prev(end())` |
| `size()` | sized_sentinel_for | `end() - begin()` |
| `operator[]` | random_access | `begin()[n]` |
| `data()` | contiguous | `to_address(begin())` |

### Minimum Requirements for a Custom View

1. **Derive from `view_interface<YourView>`** (CRTP)
2. **Provide `begin()` and `end()`** — can return different types (iterator + sentinel)
3. **Must be O(1) copyable** (views are cheap to copy)
4. **Must be default-constructible** (moved/copied views need it)

### Range Adaptor Closure Objects (C++23)

C++23 adds `std::ranges::range_adaptor_closure<Derived>` for creating pipe-compatible adaptors:

```cpp

struct my_adaptor : std::ranges::range_adaptor_closure<my_adaptor> {
    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const { return my_view(std::forward<R>(r)); }
};
// Now works with pipe: data | my_adaptor{}

```

---

## Self-Assessment

### Q1: Implement a custom `every_nth_view` that yields every Nth element of a range

```cpp

#include <iostream>
#include <iterator>
#include <ranges>
#include <vector>

template<std::ranges::forward_range R>
class every_nth_view : public std::ranges::view_interface<every_nth_view<R>> {
    R base_;
    std::size_t step_;

    class iterator {
        std::ranges::iterator_t<R> current_;
        std::ranges::sentinel_t<R> end_;
        std::size_t step_;

    public:
        using value_type = std::ranges::range_value_t<R>;
        using difference_type = std::ranges::range_difference_t<R>;

        iterator() = default;
        iterator(std::ranges::iterator_t<R> pos, std::ranges::sentinel_t<R> end, std::size_t step)
            : current_(pos), end_(end), step_(step) {}

        const auto& operator*() const { return *current_; }
        auto& operator*() { return *current_; }

        iterator& operator++() {
            for (std::size_t i = 0; i < step_ && current_ != end_; ++i)
                ++current_;
            return *this;
        }

        iterator operator++(int) { auto tmp = *this; ++*this; return tmp; }

        bool operator==(const iterator& other) const { return current_ == other.current_; }
        bool operator==(std::ranges::sentinel_t<R> s) const { return current_ == s; }
    };

public:
    every_nth_view() = default;
    every_nth_view(R base, std::size_t step) : base_(std::move(base)), step_(step) {}

    auto begin() { return iterator(std::ranges::begin(base_), std::ranges::end(base_), step_); }
    auto end()   { return std::ranges::end(base_); }
};

// Deduction guide
template<typename R>
every_nth_view(R&&, std::size_t) -> every_nth_view<std::views::all_t<R>>;

int main() {
    std::vector<int> data = {0, 1, 2, 3, 4, 5, 6, 7, 8, 9};

    auto view = every_nth_view(data, 3);

    std::cout << "Every 3rd: ";
    for (int x : view)
        std::cout << x << ' ';
    std::cout << '\n';

    // view_interface provides empty() and operator bool for free
    std::cout << "Empty? " << std::boolalpha << view.empty() << '\n';
    std::cout << "Bool?  " << static_cast<bool>(view) << '\n';
}
// Expected output:
// Every 3rd: 0 3 6 9
// Empty? false
// Bool?  true

```

**How this works:**

- `every_nth_view` inherits from `view_interface`, getting `empty()`, `operator bool`, and `front()` for free.
- The custom `iterator` advances by `step_` positions on each `++`.
- The sentinel comparison allows early termination when the iterator reaches the end.
- The deduction guide ensures `every_nth_view(data, 3)` wraps `data` in `views::all`.

### Q2: Use `view_interface` to get default implementations of `empty`, `size`, `front`, `back`

```cpp

#include <iostream>
#include <ranges>
#include <vector>

// A simple view over a contiguous range with bounds checking
template<typename T>
class span_view : public std::ranges::view_interface<span_view<T>> {
    T* data_ = nullptr;
    std::size_t size_ = 0;

public:
    span_view() = default;
    span_view(T* data, std::size_t size) : data_(data), size_(size) {}

    // Provide begin() and end() — view_interface does the rest
    T* begin() const { return data_; }
    T* end()   const { return data_ + size_; }
};

int main() {
    std::vector<int> v = {10, 20, 30, 40, 50};
    span_view sv(v.data(), v.size());

    // All of these come from view_interface for free:
    std::cout << "empty(): " << std::boolalpha << sv.empty() << '\n';
    std::cout << "size():  " << sv.size() << '\n';
    std::cout << "front(): " << sv.front() << '\n';
    std::cout << "back():  " << sv.back() << '\n';
    std::cout << "[2]:     " << sv[2] << '\n';
    std::cout << "data():  " << (sv.data() == v.data()) << '\n';

    // operator bool
    if (sv) std::cout << "View is non-empty\n";

    // Empty view
    span_view<int> empty_sv;
    std::cout << "Empty view: empty=" << empty_sv.empty()
              << " bool=" << static_cast<bool>(empty_sv) << '\n';
}
// Expected output:
// empty(): false
// size():  5
// front(): 10
// back():  50
// [2]:     30
// data():  1
// View is non-empty
// Empty view: empty=true bool=false

```

**How this works:**

- `span_view` provides only `begin()` and `end()` (returning raw pointers).
- Since raw pointers are **contiguous, random-access** iterators, `view_interface` deduces:
  - `empty()` from `begin() == end()`
  - `size()` from `end() - begin()` (sized_sentinel_for)
  - `front()` from `*begin()`
  - `back()` from `*prev(end())`
  - `operator[]` from `begin()[n]`
  - `data()` from `to_address(begin())`
- This is the maximum benefit from `view_interface`—all 7 operations for free.

### Q3: Compare implementing a custom range adaptor before and after C++23 range adaptor closure objects

**Pre-C++23: Manual pipe operator overload**

```cpp

#include <iostream>
#include <ranges>
#include <vector>

// Custom adaptor: double each element
struct double_adaptor_fn {
    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const {
        return std::forward<R>(r)
            | std::views::transform([](auto x) { return x * 2; });
    }

    // Pre-C++23: manual pipe operator
    template<std::ranges::viewable_range R>
    friend auto operator|(R&& r, const double_adaptor_fn& self) {
        return self(std::forward<R>(r));
    }
};

inline constexpr double_adaptor_fn doubled{};

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // Pre-C++23 adaptor with manual pipe
    auto result = data | doubled;
    std::cout << "Doubled: ";
    for (int x : result) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Doubled: 2 4 6 8 10

```

**C++23: Using `range_adaptor_closure`**

```cpp

#include <iostream>
#include <ranges>
#include <vector>

// C++23: inherit from range_adaptor_closure for automatic pipe support
struct doubled_closure : std::ranges::range_adaptor_closure<doubled_closure> {
    template<std::ranges::viewable_range R>
    auto operator()(R&& r) const {
        return std::forward<R>(r)
            | std::views::transform([](auto x) { return x * 2; });
    }
};

inline constexpr doubled_closure doubled23{};

// Parameterized adaptor (takes an argument)
struct multiply_by_closure {
    int factor;

    struct closure : std::ranges::range_adaptor_closure<closure> {
        int factor;
        explicit closure(int f) : factor(f) {}

        template<std::ranges::viewable_range R>
        auto operator()(R&& r) const {
            return std::forward<R>(r)
                | std::views::transform([f = factor](auto x) { return x * f; });
        }
    };

    auto operator()(int f) const { return closure{f}; }
};

inline constexpr multiply_by_closure multiply_by{};

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5};

    // C++23 adaptor — pipe works automatically
    auto result = data | doubled23;
    std::cout << "Doubled: ";
    for (int x : result) std::cout << x << ' ';
    std::cout << '\n';

    // Parameterized adaptor
    auto tripled = data | multiply_by(3);
    std::cout << "Tripled: ";
    for (int x : tripled) std::cout << x << ' ';
    std::cout << '\n';

    // Compose adaptors (C++23 enables chaining)
    auto pipeline = doubled23 | multiply_by(3);  // double then triple = 6x
    auto result6x = data | pipeline;
    std::cout << "6x: ";
    for (int x : result6x) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// Doubled: 2 4 6 8 10
// Tripled: 3 6 9 12 15
// 6x: 6 12 18 24 30

```

**Comparison:**

| Aspect | Pre-C++23 | C++23 with `range_adaptor_closure` |
| --- | --- | --- |
| Pipe support | Manual `friend operator\|` | Automatic from base class |
| Adaptor composition | Not supported | `a \| b` creates combined adaptor |
| Boilerplate | ~10 lines for pipe operator | 0 lines — just inherit |
| Parameterized | Must handle argument binding manually | Return a closure from `operator()(args)` |

---

## Notes

- `view_interface` uses CRTP: always inherit as `view_interface<YourView>`.
- A view must be **O(1) copyable**—store only iterators/sentinels/small data, never own a container.
- Custom views should satisfy `std::ranges::enable_view<YourView>` (automatic when inheriting from `view_interface`).
- For simple adaptors, composing existing views (transform, filter, take, etc.) is usually sufficient—only write custom views when the standard adaptors can't express your logic.
