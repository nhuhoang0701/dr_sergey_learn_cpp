# Use the Empty Base Class Optimization (EBO) with [[no_unique_address]]

**Category:** Best Practices & Idioms  
**Item:** #405  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/no_unique_address>  

---

## Topic Overview

Every C++ object must have a unique address, so even an empty struct occupies at least 1 byte. **EBO** allows empty base classes to take zero space. C++20's `[[no_unique_address]]` extends this to **members**.

```cpp

struct Empty {};          // sizeof = 1 (must have unique address)
struct A : Empty { int x; };  // sizeof = 4 (EBO: Empty takes 0)

struct B {
    [[no_unique_address]] Empty e;  // takes 0 bytes (C++20)
    int x;
};  // sizeof = 4

```

---

## Self-Assessment

### Q1: Empty base class optimization in action

```cpp

#include <iostream>

struct Empty {};   // sizeof = 1
struct Tag1 {};
struct Tag2 {};

// WITHOUT EBO: empty member wastes space
struct WithoutEBO {
    Empty e;    // 1 byte + padding
    int data;   // 4 bytes
};  // total: 8 bytes (1 + padding + 4)

// WITH EBO via inheritance
struct WithEBO : Empty {
    int data;
};  // total: 4 bytes! Empty base takes 0

// Multiple empty bases
struct MultiBase : Tag1, Tag2 {
    int data;
};  // still 4 bytes

int main() {
    std::cout << "Empty:      " << sizeof(Empty) << '\n';       // 1
    std::cout << "WithoutEBO: " << sizeof(WithoutEBO) << '\n';  // 8
    std::cout << "WithEBO:    " << sizeof(WithEBO) << '\n';     // 4
    std::cout << "MultiBase:  " << sizeof(MultiBase) << '\n';   // 4

    // EBO doesn't apply when two subobjects have the same type:
    struct TwoSame : Empty {
        Empty e2;  // must have unique address from base
    };
    std::cout << "TwoSame:    " << sizeof(TwoSame) << '\n';     // 2
}
// Expected output:
// Empty:      1
// WithoutEBO: 8
// WithEBO:    4
// MultiBase:  4
// TwoSame:    2

```

### Q2: Use `[[no_unique_address]]` for member EBO

```cpp

#include <iostream>
#include <functional>  // std::less

struct Stateless {
    void operator()() const {}
};

// Old approach: inherit to get EBO
template<typename T, typename Comp>
struct OldWay : private Comp {
    T value;
};  // works, but inheritance is awkward

// C++20: [[no_unique_address]] on members
template<typename T, typename Comp = std::less<T>>
struct Container {
    [[no_unique_address]] Comp comp;  // 0 bytes if Comp is empty!
    T* data;
    size_t size;
};

// Without the attribute
template<typename T, typename Comp = std::less<T>>
struct ContainerBad {
    Comp comp;  // 1 byte + padding = wastes space
    T* data;
    size_t size;
};

int main() {
    std::cout << "std::less<int> size: " << sizeof(std::less<int>) << '\n';  // 1

    std::cout << "Container<int>:    " << sizeof(Container<int>) << '\n';     // 16
    std::cout << "ContainerBad<int>: " << sizeof(ContainerBad<int>) << '\n';  // 24

    // With a stateful comparator, both are the same size:
    struct BigComp { int state[4]; bool operator()(int, int) const { return true; } };
    std::cout << "Container<int, BigComp>:    "
              << sizeof(Container<int, BigComp>) << '\n';     // 32
    std::cout << "ContainerBad<int, BigComp>: "
              << sizeof(ContainerBad<int, BigComp>) << '\n';  // 32
}
// Expected output:
// std::less<int> size: 1
// Container<int>:    16
// ContainerBad<int>: 24
// Container<int, BigComp>:    32
// ContainerBad<int, BigComp>: 32

```

### Q3: Compressed pair using EBO (like `unique_ptr` stores deleters)

```cpp

#include <iostream>
#include <memory>

// compressed_pair: stores two values, applying EBO when possible
template<typename T1, typename T2>
struct compressed_pair {
    [[no_unique_address]] T1 first;
    [[no_unique_address]] T2 second;
};

struct EmptyAlloc {};

int main() {
    // Both empty: size = 1 (minimum)
    compressed_pair<EmptyAlloc, EmptyAlloc> p1;
    std::cout << "both empty: " << sizeof(p1) << '\n';  // 1 or 2

    // One empty, one int: size = 4
    compressed_pair<EmptyAlloc, int> p2;
    std::cout << "empty + int: " << sizeof(p2) << '\n';  // 4

    // Two ints: size = 8
    compressed_pair<int, int> p3;
    std::cout << "int + int: " << sizeof(p3) << '\n';  // 8

    // Real-world: std::unique_ptr uses this for stateless deleters
    auto p = std::make_unique<int>(42);
    std::cout << "unique_ptr<int> size: " << sizeof(p) << '\n';  // 8 (same as raw ptr!)
    // The default_delete<int> is empty and gets compressed away

    struct CustomDeleter {
        int log_id;  // stateful: takes space!
        void operator()(int* p) { delete p; }
    };
    std::unique_ptr<int, CustomDeleter> p4(new int(5), CustomDeleter{1});
    std::cout << "unique_ptr<int, CustomDeleter> size: " << sizeof(p4) << '\n';  // 16
}
// Expected output:
// both empty: 2
// empty + int: 4
// int + int: 8
// unique_ptr<int> size: 8
// unique_ptr<int, CustomDeleter> size: 16

```

---

## Notes

- `[[no_unique_address]]` is a C++20 attribute. MSVC uses `[[msvc::no_unique_address]]`.
- It only works when the member type is empty (no data members).
- The standard library uses EBO extensively: `unique_ptr`, allocators, comparators.
- Two `[[no_unique_address]]` members of the **same type** cannot share address (they need distinct identities).
