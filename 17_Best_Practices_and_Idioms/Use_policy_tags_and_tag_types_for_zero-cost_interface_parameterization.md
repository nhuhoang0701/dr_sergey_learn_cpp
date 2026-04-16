# Use policy tags and tag types for zero-cost interface parameterization

**Category:** Best Practices & Idioms  
**Item:** #296  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>  

---

## Topic Overview

**Tag types** are empty structs used to select overloads or policies at compile time. They produce zero-overhead code because the empty struct is optimized away completely.

### Standard Library Examples

```cpp

std::piecewise_construct_t   // tag for std::pair
std::in_place_t              // tag for std::optional, std::variant
std::allocator_arg_t         // tag for allocator-aware types
std::defer_lock_t            // tag for std::unique_lock

```

---

## Self-Assessment

### Q1: Define policy tags and overload on them

```cpp

#include <cstddef>
#include <iostream>
#include <stdexcept>
#include <vector>

// Policy tags (empty structs)
struct check_bounds_t {};
struct no_check_t {};

// Tag instances (inline constexpr for ODR safety)
inline constexpr check_bounds_t check_bounds{};
inline constexpr no_check_t no_check{};

template<typename T>
class Array {
    std::vector<T> data_;
public:
    Array(std::initializer_list<T> init) : data_(init) {}

    // Overload 1: with bounds checking
    T& operator()(size_t i, check_bounds_t) {
        if (i >= data_.size())
            throw std::out_of_range("index " + std::to_string(i));
        return data_[i];
    }

    // Overload 2: no bounds checking (zero overhead)
    T& operator()(size_t i, no_check_t) noexcept {
        return data_[i];
    }

    size_t size() const { return data_.size(); }
};

int main() {
    Array<int> arr{10, 20, 30, 40, 50};

    // Safe access (debug/testing)
    std::cout << arr(2, check_bounds) << '\n';  // 30

    // Fast access (hot path)
    std::cout << arr(4, no_check) << '\n';       // 50

    try {
        arr(10, check_bounds);  // throws!
    } catch (const std::out_of_range& e) {
        std::cout << "Caught: " << e.what() << '\n';
    }
}
// Expected output:
// 30
// 50
// Caught: index 10

```

### Q2: Show that policy tags generate separate code paths with zero runtime branching

```cpp

#include <iostream>

// Policy tags
struct sorted_t {};
struct unsorted_t {};
inline constexpr sorted_t sorted{};
inline constexpr unsorted_t unsorted{};

// Two completely different algorithms, selected at compile time
template<typename Iter>
bool find(Iter begin, Iter end, int value, sorted_t) {
    // Binary search for sorted data — O(log n)
    std::cout << "[binary search] ";
    auto it = std::lower_bound(begin, end, value);
    return it != end && *it == value;
}

template<typename Iter>
bool find(Iter begin, Iter end, int value, unsorted_t) {
    // Linear search for unsorted data — O(n)
    std::cout << "[linear search] ";
    return std::find(begin, end, value) != end;
}

int main() {
    int sorted_data[] = {1, 3, 5, 7, 9, 11};
    int unsorted_data[] = {7, 2, 9, 1, 5, 3};

    // Compiler generates DIFFERENT code for each call:
    bool r1 = find(std::begin(sorted_data), std::end(sorted_data), 7, sorted);
    std::cout << std::boolalpha << r1 << '\n';

    bool r2 = find(std::begin(unsorted_data), std::end(unsorted_data), 5, unsorted);
    std::cout << r2 << '\n';

    // NO runtime if/else branching! The tag selects the overload
    // at compile time, and the empty tag parameter is optimized away.
}
// Expected output:
// [binary search] true
// [linear search] true

```

### Q3: Compare policy tags with template boolean parameters

```cpp

#include <iostream>
#include <vector>

// Approach 1: bool template parameter (bad readability)
template<bool CheckBounds>
int access_v1(const std::vector<int>& v, size_t i) {
    if constexpr (CheckBounds) {
        return v.at(i);
    } else {
        return v[i];
    }
}

// Approach 2: tag types (good readability)
struct checked_t {};
struct unchecked_t {};
inline constexpr checked_t checked{};
inline constexpr unchecked_t unchecked{};

int access_v2(const std::vector<int>& v, size_t i, checked_t) { return v.at(i); }
int access_v2(const std::vector<int>& v, size_t i, unchecked_t) { return v[i]; }

int main() {
    std::vector<int> v = {10, 20, 30};

    // BAD: what does true/false mean?
    access_v1<true>(v, 1);   // checked? sorted? reversed?
    access_v1<false>(v, 1);  // who knows without looking up the definition

    // GOOD: self-documenting
    access_v2(v, 1, checked);    // obviously: bounds-checked
    access_v2(v, 1, unchecked);  // obviously: no bounds check

    std::cout << access_v2(v, 1, checked) << '\n';
}
// Expected output:
// 20

```

| Feature | `template<bool B>` | Tag types |
| --- | --- | --- |
| Readability | `func<true>(x)` — unclear | `func(x, checked)` — clear |
| Can mix with overloads | Awkward | Natural |
| Runtime overhead | Zero | Zero |
| Extensible | No (only true/false) | Yes (add new tags) |
| Error messages | Template errors | "no matching overload" |

---

## Notes

- Tag types cost zero bytes at runtime — they're empty structs, optimized away.
- Use `inline constexpr` tag instances in headers for ODR safety.
- The STL uses tags extensively: `std::piecewise_construct`, `std::in_place`, `std::defer_lock`.
- Combine with concepts for even better error messages in C++20.
