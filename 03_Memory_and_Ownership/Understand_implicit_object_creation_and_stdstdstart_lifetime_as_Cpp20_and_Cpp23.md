# Understand implicit object creation and std::start_lifetime_as (C++20/C++23)

**Category:** Memory and Ownership  
**Standard:** C++23  
**Reference:** <https://wg21.link/P0593> <https://wg21.link/P2590>  

---

## Topic Overview

C++ has strict rules about object lifetimes. Casting raw memory to a type without properly constructing an object is UB. This sounds pedantic until you realize it affects some of the most common low-level patterns in systems programming - reading network packets, memory-mapped files, and serialization buffers. C++20 introduced implicit object creation to bless those patterns, and C++23 added `std::start_lifetime_as` to make the intent explicit.

### The Pre-C++20 Problem

Before C++20, interpreting raw bytes as a typed struct through `reinterpret_cast` was technically undefined behavior even when the byte layout was correct. The abstract machine had no `Header` object at that address - just bytes. That distinction matters to the optimizer.

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

The reason this trips people up is that it usually "works" in practice, because compilers are reluctant to optimize away the access. But "usually works" and "well-defined" are two different things, and the gap between them can bite you on a future compiler version or at a higher optimization level.

### C++20: Implicit Object Creation

C++20 resolved this for the most common cases by specifying that certain operations - `malloc`, `operator new`, `memcpy`, and `memmove` - **implicitly create objects** of implicit-lifetime types in the destination storage. If `Header` is an implicit-lifetime type (trivially copyable, trivially destructible), you now have a well-defined object after the allocation.

```cpp
#include <cstdlib>
#include <cstring>

// C++20: malloc, operator new, memcpy, and memmove
// implicitly create objects of implicit-lifetime types.

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

Notice the cast changed from `reinterpret_cast` to `static_cast` - that is intentional. The implicit creation makes the static cast valid.

### C++23: std::start_lifetime_as

C++23 went further and gave you an explicit, readable spell for the same operation. `std::start_lifetime_as` both creates the object and returns a correctly-typed pointer in one call. It is a no-op at runtime - the instruction stream does not change - but it informs the abstract machine that a lifetime has begun, making the access well-defined.

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

Scalar types, arrays thereof, and class types that have a trivial destructor and at least one trivial constructor (default, copy, or move). Essentially: types whose construction/destruction is a no-op. `std::string` is NOT implicit-lifetime because its constructor and destructor do real work (heap allocation/deallocation).

### Q2: Why was this necessary

High-performance serialization, memory-mapped I/O, and network protocol parsing all need to interpret raw bytes as typed objects. Before C++20, doing this via `reinterpret_cast` was technically UB (even for trivially-copyable types). The standard needed to bless these patterns so that the enormous amount of existing code doing this was formally correct, and so that compilers could not legally optimize it away.

### Q3: What's the difference between `start_lifetime_as` and `reinterpret_cast`

`reinterpret_cast` changes the type of a pointer but does NOT create an object. `start_lifetime_as` both creates an object AND returns a correctly-typed pointer. Only `start_lifetime_as` makes the program well-defined. Think of it this way: `reinterpret_cast` is a lie you tell the type system; `start_lifetime_as` is a promise you make to the abstract machine.

---

## Notes

- `start_lifetime_as<T>` requires the storage to be properly aligned for `T`.
- It's a no-op at runtime - it exists purely for the abstract machine's object model.
- Essential for safe zero-copy network/file parsing.
- `std::bit_cast` copies bytes (creates a new object); `start_lifetime_as` reinterprets in-place.
