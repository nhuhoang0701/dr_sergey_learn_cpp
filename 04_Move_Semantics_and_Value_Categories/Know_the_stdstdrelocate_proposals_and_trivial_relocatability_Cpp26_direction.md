# Know the std::relocate proposals and trivial relocatability (C++26 direction)

**Category:** Move Semantics and Value Categories  
**Standard:** C++26 (proposed)  
**Reference:** <https://wg21.link/P1144>  

---

## Topic Overview

**Relocation** = move-construct + destroy-source, combined into one operation. For many types ("trivially relocatable" types), this can be optimized to a simple `memcpy`.

### What is Trivial Relocatability

```cpp

#include <cstring>

// A type is trivially relocatable if moving it + destroying the source
// is equivalent to memcpy.

struct TriviallyRelocatable {
    int x;
    double y;
    std::unique_ptr<int> ptr;  // unique_ptr is trivially relocatable!
    // Move = copy bytes, destroy source = no-op
    // So: relocate = memcpy
};

// NOT trivially relocatable:
struct SelfReferential {
    int data;
    int* self_ptr = &data;  // Points to itself!
    // After memcpy, self_ptr points to OLD location — broken!
};

```

### Why It Matters for vector

```cpp

// vector::push_back with reallocation currently:
// 1. Allocate new buffer
// 2. Move-construct each element to new buffer  (N move constructors)
// 3. Destroy each element in old buffer          (N destructors)
// 4. Deallocate old buffer

// With trivial relocatability:
// 1. Allocate new buffer
// 2. memcpy(new_buf, old_buf, N * sizeof(T))   (ONE memory operation)
// 3. Deallocate old buffer (no destructors needed)
// Speedup: ~5-10x for vector reallocation of trivially relocatable types

```

### The Proposed API

```cpp

// P1144 proposal:
template<typename T>
concept trivially_relocatable = /* compiler-determined */;

// Relocate = move + destroy, optimized when possible:
template<typename T>
T* relocate_at(T* source, T* dest) {
    if constexpr (std::is_trivially_relocatable_v<T>) {
        std::memcpy(dest, source, sizeof(T));
    } else {
        std::construct_at(dest, std::move(*source));
        std::destroy_at(source);
    }
    return dest;
}

```

---

## Self-Assessment

### Q1: Which standard library types are trivially relocatable

`unique_ptr`, `shared_ptr`, `string` (with SSO), `vector`, `optional`, `variant`, and most containers. The only common non-trivially-relocatable type is one with self-referential pointers or registered in a global data structure.

### Q2: Why can't the compiler auto-detect trivial relocatability

The compiler can't see through `virtual` functions, external libraries, or types that register themselves in constructors. A type might store `this` in a global registry — memcpy would leave a stale entry. An explicit `[[trivially_relocatable]]` attribute is needed.

### Q3: How do existing libraries handle this

Folly (Facebook) uses `folly::IsRelocatable<T>` trait and `fbvector` which does `memcpy` reallocation. Abseil has similar optimizations. BSL (Bloomberg) uses `bslmf::IsBitwiseMoveable`. All pre-date the standard proposal.

---

## Notes

- P1144 has been in discussion since 2018 and targets C++26.
- `[[trivially_relocatable]]` attribute opt-in for user types.
- The main beneficiary is `std::vector` — reallocation becomes O(1) memcpy.
- MSVC STL already uses `_Is_trivially_relocatable` internally.
