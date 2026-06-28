# Know the rules for [[no_unique_address]] and its effect on class layout

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>  

---

## Topic Overview

`[[no_unique_address]]` allows empty class members to occupy zero bytes, enabling the Empty Base Optimization (EBO) for members instead of just bases. It tells the compiler:
> "This member has no state and doesn't need its own unique address. You can optimize it away entirely."

### The Problem

C++ requires every distinct object to have a unique address - even an empty one. That means an `Empty` member still burns at least one byte (plus alignment padding). Before C++20, the only way around this was inheritance:

```cpp
#include <iostream>
#include <functional>
#include <memory>

struct Empty {};  // sizeof(Empty) == 1 (due to unique address requirement)

struct WithoutAttr {
    int value;
    Empty tag;         // Takes 1 byte + padding = 4 bytes wasted
};

struct WithAttr {
    int value;
    [[no_unique_address]] Empty tag;  // both the member and the padding are eliminated
};

int main() {
    std::cout << sizeof(WithoutAttr) << "\n"; // 8 (int + padding for Empty)
    std::cout << sizeof(WithAttr) << "\n";    // 4 (Empty occupies no space)
}
```

With the attribute, the empty member is marked as potentially-overlapping and may be optimized to zero bytes.

### When "Overlap" Actually Matters
The "overlap" language from the standard becomes relevant in more complex cases:
```cpp
struct A { int x; };  // 4 bytes

struct B {
    [[no_unique_address]] A a;  // 4 bytes
    char c;                      // Can be placed in A's tail padding!
};
// sizeof(B) = 8 (not 16)
```

Here, c can be placed in A's tail padding because a is marked as potentially-overlapping. But this is about overlapping with another member's padding, not "the empty member's storage with surrounding padding."

### Practical Use: Stateless Allocators and Deleters

The most common real-world use is in smart pointer and container implementations, where the deleter or allocator is often a stateless empty type:

```cpp
#include <memory>
#include <iostream>

template<typename T, typename Deleter = std::default_delete<T>>
class SmartPtr {
    T* ptr_;
    [[no_unique_address]] Deleter deleter_;
public:
    SmartPtr(T* p, Deleter d = {}) : ptr_(p), deleter_(std::move(d)) {}
    ~SmartPtr() { if (ptr_) deleter_(ptr_); }
    T& operator*() const { return *ptr_; }
};

int main() {
    // std::default_delete is empty - takes zero extra space
    std::cout << sizeof(SmartPtr<int>) << "\n"; // Same as sizeof(int*)
    // Without [[no_unique_address]], it would be sizeof(int*) + padding
}
```

### Rules and Limitations

Two members of the same type still need distinct addresses, so they can't overlap even with the attribute. Different types, however, can:

```cpp
struct TwoEmpty {
    [[no_unique_address]] Empty a;
    [[no_unique_address]] Empty b;
    // a and b CANNOT overlap if they have the same type
    // (each object of the same type needs a unique address)
};
// sizeof(TwoEmpty) == 2 (not 0, not 1)

struct DifferentEmpty {
    struct Tag1 {};
    struct Tag2 {};
    [[no_unique_address]] Tag1 a;
    [[no_unique_address]] Tag2 b;
    // Different types CAN overlap
};
// sizeof(DifferentEmpty) == 1
```

If you need two empty members of the same type without the size penalty, derive from one and use the attribute on the other, or use distinct tag types.

---

## Self-Assessment

### Q1: When can [[no_unique_address]] members overlap

Two `[[no_unique_address]]` members can overlap if they have different types. Two members of the same type cannot overlap because C++ requires distinct objects of the same type to have distinct addresses (`&a != &b`).

### Q2: How does this relate to the Empty Base Optimization

EBO allows empty base classes to take zero bytes. `[[no_unique_address]]` extends this to members. Before C++20, the only way to get zero-cost empty types was inheritance (compressed pair pattern). Now you can use a member attribute instead.

### Q3: What is the MSVC caveat

MSVC (as of VS 2022) uses `[[msvc::no_unique_address]]` instead of the standard attribute due to ABI compatibility concerns. The standard attribute is accepted but ignored. Use a macro to write portable code:

```cpp
#ifdef _MSC_VER
  #define NO_UNIQUE_ADDRESS [[msvc::no_unique_address]]
#else
  #define NO_UNIQUE_ADDRESS [[no_unique_address]]
#endif
```

---

## Notes

- Essential for reducing `sizeof` in allocator-aware containers, smart pointers, and policy-based designs.
- `std::unique_ptr` uses EBO/`[[no_unique_address]]` internally for the deleter.
- MSVC requires `[[msvc::no_unique_address]]` - the standard attribute is a no-op on MSVC.
- Combined with `std::is_empty_v<T>` for conditional optimization.
