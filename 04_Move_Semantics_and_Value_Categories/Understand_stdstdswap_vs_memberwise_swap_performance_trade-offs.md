# Understand std::swap vs memberwise swap performance trade-offs

**Category:** Move Semantics and Value Categories  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/swap>  

---

## Topic Overview

`std::swap` uses move construction + 2 move assignments (3 moves total). A custom memberwise swap can be more efficient for types with many members.

### std::swap Implementation

```cpp

// Standard library std::swap:
template<typename T>
void swap(T& a, T& b)
    noexcept(std::is_nothrow_move_constructible_v<T> &&
             std::is_nothrow_move_assignable_v<T>) {
    T temp = std::move(a);  // 1 move construction
    a = std::move(b);       // 1 move assignment
    b = std::move(temp);    // 1 move assignment
    // Total: 3 moves, each moving ALL members
}

```

### Memberwise Swap

```cpp

#include <string>
#include <vector>
#include <algorithm>

class Document {
    std::string title_;
    std::string author_;
    std::vector<std::string> pages_;
    int version_ = 0;

public:
    // Custom swap: swaps each member individually
    friend void swap(Document& a, Document& b) noexcept {
        using std::swap;
        swap(a.title_, b.title_);   // string::swap — O(1) pointer swap
        swap(a.author_, b.author_); // string::swap — O(1) pointer swap
        swap(a.pages_, b.pages_);   // vector::swap — O(1) pointer swap
        swap(a.version_, b.version_); // int swap
    }
    // Each member's swap is optimized (no temporary string/vector created)
    // vs std::swap<Document>: creates temporary Document (copies all 4 members)
};

```

### Performance Comparison

```cpp

std::swap<Document>:

  1. temp = move(a)     // Move 3 heap pointers + int
  2. a = move(b)        // Move 3 heap pointers + int  
  3. b = move(temp)     // Move 3 heap pointers + int

  Total: 9 pointer moves + 3 int moves = 12 operations

memberwise swap:
  4 individual swaps, each is ~3 pointer-sized operations
  Total: ~12 operations (similar)

BUT: for types where move is expensive (e.g., stateful allocators,
non-movable members), memberwise swap wins significantly.

```

---

## Self-Assessment

### Q1: When is custom swap necessary

When the type has non-movable members, uses custom allocators (swap may need to swap allocators), or when you need stronger exception safety guarantees than 3 moves provide.

### Q2: Why should swap always be noexcept

Swap is a fundamental operation used by `std::sort`, `std::partition`, and container operations. If swap can throw, these algorithms lose their exception safety guarantees. `noexcept` swap enables `vector::push_back` to use move instead of copy.

### Q3: How does ADL find custom swap

```cpp

template<typename T>
void generic_swap(T& a, T& b) {
    using std::swap;    // Bring std::swap into scope as fallback
    swap(a, b);         // Unqualified call — ADL finds custom swap first
}
// If Document has a friend swap, ADL finds it.
// Otherwise, std::swap is used as fallback.

```

---

## Notes

- Always provide `friend void swap(T&, T&) noexcept` for types with heap resources.
- Use the `using std::swap; swap(a, b);` pattern in generic code.
- The copy-and-swap idiom uses custom swap for exception-safe assignment.
- For trivially-copyable types, `std::swap` compiles to optimal `xchg` instructions.
