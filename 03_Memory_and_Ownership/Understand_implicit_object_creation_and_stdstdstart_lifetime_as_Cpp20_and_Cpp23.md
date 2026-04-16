# Understand implicit object creation and std::start_lifetime_as (C++20/C++23)

**Category:** Memory and Ownership  
**Standard:** C++23  
**Reference:** <https://wg21.link/P0593> <https://wg21.link/P2590>  

---

## Topic Overview

C++ has strict rules about object lifetimes. Casting raw memory to a type without properly constructing an object is UB. C++20 introduced implicit object creation, and C++23 added `std::start_lifetime_as` to make common patterns well-defined.

### The Pre-C++20 Problem

```cpp

#include <cstdlib>
#include <cstring>

struct Header {
    uint32_t magic;
    uint32_t size;
};

// Pre-C++20: technically UB!
void parse_bad(const char* buffer) {
    auto* header = reinterpret_cast<const Header*>(buffer);
    // UB: no Header object exists at that address
    // Even though the bytes are correct, there's no object
    if (header->magic == 0xDEADBEEF) { /* ... */ }
}

```

### C++20: Implicit Object Creation

```cpp

#include <cstdlib>
#include <cstring>

// C++20: malloc, operator new, memcpy, and memmove
// implicitly create objects in the destination storage.

void parse_c20(size_t size) {
    void* storage = std::malloc(size);
    // malloc implicitly creates objects of implicit-lifetime types
    auto* header = static_cast<Header*>(storage);
    // OK in C++20: Header is implicit-lifetime, malloc creates it
    // (Header must be trivially copyable and trivially destructible)
}

// Implicit-lifetime types:
// - Scalar types (int, double, pointers)
// - Arrays of implicit-lifetime types
// - Aggregates of implicit-lifetime types with trivial destructor

```

### C++23: std::start_lifetime_as

```cpp

#include <memory>

// C++23: explicit, readable way to begin object lifetime
void parse_c23(const unsigned char* buffer) {
    auto* header = std::start_lifetime_as<Header>(buffer);
    // Creates a Header object at buffer's address
    // Returns const Header* (preserves constness)
    // The bytes in buffer are reused as-is
}

// Also: start_lifetime_as_array
void parse_array(const unsigned char* buffer, size_t count) {
    auto* items = std::start_lifetime_as_array<uint32_t>(buffer, count);
    // Creates an array of count uint32_t objects
}

```

---

## Self-Assessment

### Q1: What types qualify as implicit-lifetime types

Scalar types, arrays thereof, and class types that have a trivial destructor and at least one trivial constructor (default, copy, or move). Essentially: types whose construction/destruction is a no-op. `std::string` is NOT implicit-lifetime.

### Q2: Why was this necessary

High-performance serialization, memory-mapped I/O, and network protocol parsing all need to interpret raw bytes as typed objects. Before C++20, doing this via `reinterpret_cast` was technically UB (even for trivially-copyable types). The standard needed to bless these patterns.

### Q3: What's the difference between `start_lifetime_as` and `reinterpret_cast`

`reinterpret_cast` changes the type of a pointer but does NOT create an object. `start_lifetime_as` both creates an object AND returns a correctly-typed pointer. Only `start_lifetime_as` makes the program well-defined.

---

## Notes

- `start_lifetime_as<T>` requires the storage to be properly aligned for `T`.
- It's a no-op at runtime — it exists purely for the abstract machine's object model.
- Essential for safe zero-copy network/file parsing.
- `std::bit_cast` copies bytes (creates a new object); `start_lifetime_as` reinterprets in-place.
