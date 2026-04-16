# Know the rules for [[no_unique_address]] and its effect on class layout

**Category:** Core Language Fundamentals  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>  

---

## Topic Overview

`[[no_unique_address]]` allows empty class members to occupy zero bytes, enabling the Empty Base Optimization (EBO) for members instead of just bases.

### The Problem

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
    [[no_unique_address]] Empty tag;  // Overlaps with padding — zero waste
};

int main() {
    std::cout << sizeof(WithoutAttr) << "\n"; // 8 (int + padding for Empty)
    std::cout << sizeof(WithAttr) << "\n";    // 4 (Empty occupies no space)
}

```

### Practical Use: Stateless Allocators and Deleters

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
    // std::default_delete is empty — takes zero extra space
    std::cout << sizeof(SmartPtr<int>) << "\n"; // Same as sizeof(int*)
    // Without [[no_unique_address]], it would be sizeof(int*) + padding
}

```

### Rules and Limitations

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

---

## Self-Assessment

### Q1: When can [[no_unique_address]] members overlap

Two `[[no_unique_address]]` members can overlap if they have different types. Two members of the same type cannot overlap because C++ requires distinct objects of the same type to have distinct addresses (`&a != &b`).

### Q2: How does this relate to the Empty Base Optimization

EBO allows empty base classes to take zero bytes. `[[no_unique_address]]` extends this to members. Before C++20, the only way to get zero-cost empty types was inheritance (compressed pair pattern). Now you can use a member attribute instead.

### Q3: What is the MSVC caveat

MSVC (as of VS 2022) uses `[[msvc::no_unique_address]]` instead of the standard attribute due to ABI compatibility concerns. The standard attribute is accepted but ignored. Use a macro: `#ifdef _MSC_VER #define NO_UNIQUE_ADDRESS [[msvc::no_unique_address]] ...`

---

## Notes

- Essential for reducing `sizeof` in allocator-aware containers, smart pointers, and policy-based designs.
- `std::unique_ptr` uses EBO/`[[no_unique_address]]` internally for the deleter.
- MSVC requires `[[msvc::no_unique_address]]` — the standard attribute is a no-op on MSVC.
- Combined with `std::is_empty_v<T>` for conditional optimization.
