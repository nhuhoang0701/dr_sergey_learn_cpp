# Use std::pmr::vector for arena-allocated vectors without template parameter changes

**Category:** Standard Library — Containers  
**Item:** #288  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/polymorphic_allocator>  

---

## Topic Overview

`std::pmr::vector` is a type alias for `std::vector<T, std::pmr::polymorphic_allocator<T>>`. It uses **runtime polymorphism** to select different memory resources without changing the vector's type. This enables arena allocation, stack-based buffers, and custom memory strategies without the template parameter explosion of traditional allocators.

### The Problem with Classic Allocators

```cpp

// Classic approach: allocator is part of the TYPE
std::vector<int, MyArenaAllocator<int>> arena_vec;
std::vector<int, std::allocator<int>> default_vec;

// These are DIFFERENT TYPES — can't be assigned to each other!
// arena_vec = default_vec;  // ERROR: type mismatch

// Every function that takes this vector must be templated:
template <typename Alloc>
void process(std::vector<int, Alloc>& v);  // annoying

```

### The PMR Solution

```cpp

#include <memory_resource>
#include <vector>

// pmr::vector<int> is ALWAYS the same type, regardless of memory resource
std::pmr::vector<int> v1(&my_arena);      // uses arena
std::pmr::vector<int> v2;                  // uses default resource
// Same type! Can pass to the same function:
void process(std::pmr::vector<int>& v);    // works for both

```

### Memory Resource Hierarchy

```cpp

std::pmr::memory_resource (abstract base)
├── new_delete_resource()           — wraps global new/delete
├── null_memory_resource()          — always throws bad_alloc
├── monotonic_buffer_resource       — fast bump allocator (no dealloc)
├── unsynchronized_pool_resource    — pool allocator (single-threaded)
├── synchronized_pool_resource      — pool allocator (thread-safe)
└── Your custom memory_resource     — override allocate/deallocate/is_equal

```

### Key Relationships

| Standard allocator | PMR equivalent |
| --- | --- |
| `std::vector<T>` | `std::pmr::vector<T>` |
| `std::string` | `std::pmr::string` |
| `std::map<K,V>` | `std::pmr::map<K,V>` |
| `std::unordered_map<K,V>` | `std::pmr::unordered_map<K,V>` |

---

## Self-Assessment

### Q1: Create a pmr::vector backed by a monotonic_buffer_resource and verify no heap allocation

```cpp

#include <iostream>
#include <memory_resource>
#include <vector>
#include <cstddef>

// === Custom resource that detects heap fallback ===
class TrackingResource : public std::pmr::memory_resource {
    bool heap_used_ = false;
protected:
    void* do_allocate(size_t bytes, size_t alignment) override {
        heap_used_ = true;
        std::cout << "  [HEAP] Allocated " << bytes << " bytes\n";
        return ::operator new(bytes, std::align_val_t(alignment));
    }
    void do_deallocate(void* p, size_t bytes, size_t alignment) override {
        ::operator delete(p, bytes, std::align_val_t(alignment));
    }
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
public:
    bool heap_used() const { return heap_used_; }
};

int main() {
    // === Stack buffer: 1 KB on the stack ===
    alignas(alignof(std::max_align_t)) std::byte buffer[1024];

    // === monotonic_buffer_resource ===
    // - Allocates by bumping a pointer forward (very fast)
    // - NEVER deallocates individual objects
    // - Falls back to upstream resource if buffer is exhausted
    TrackingResource fallback;
    std::pmr::monotonic_buffer_resource arena(buffer, sizeof(buffer), &fallback);

    // === pmr::vector using the arena ===
    std::pmr::vector<int> v(&arena);

    // Push elements — all allocations come from the stack buffer
    for (int i = 0; i < 50; ++i)
        v.push_back(i);

    std::cout << "Vector size: " << v.size() << "\n";
    std::cout << "Heap used: " << (fallback.heap_used() ? "YES" : "NO") << "\n";
    // Output:
    // Vector size: 50
    // Heap used: NO

    // === Exceed the buffer to trigger fallback ===
    std::cout << "\nFilling beyond stack buffer:\n";
    for (int i = 0; i < 500; ++i)
        v.push_back(i);
    std::cout << "Heap used now: " << (fallback.heap_used() ? "YES" : "NO") << "\n";
    // Output:
    // [HEAP] Allocated XXXX bytes
    // Heap used now: YES

    // === Monotonic: no per-element deallocation ===
    // Memory is freed when the monotonic_buffer_resource is destroyed.
    // This is why it's so fast — just bump a pointer.

    return 0;
}
// When `arena` goes out of scope, it releases any upstream allocations.
// The stack buffer is automatically reclaimed.

```

**How it works:**

- `monotonic_buffer_resource` takes a stack buffer and an optional upstream resource for overflow.
- Allocation is a simple pointer bump — O(1), no fragmentation, cache-friendly.
- Individual `deallocate()` calls are no-ops. All memory is released at once when the resource is destroyed.
- The `TrackingResource` fallback detects if the stack buffer was exceeded.
- **Use case:** Short-lived allocations in a function scope (e.g., building a temporary data structure, parsing).

### Q2: Compare pmr::vector<T> and vector<T, MyAllocator> for ergonomics and binary compatibility

| Aspect | `vector<T, MyAllocator>` | `pmr::vector<T>` |
| --- | --- | --- |
| **Type identity** | Different type per allocator | Always the same type |
| **ABI/Binary compat** | Different types → different mangled names → can't cross library boundaries | Same type everywhere → stable ABI |
| **Function signatures** | Must template on allocator: `template<class A> void f(vector<T,A>&)` | Single signature: `void f(pmr::vector<T>&)` |
| **Virtual dispatch** | No (allocator is a template parameter, inlined) | Yes (memory_resource is a virtual base → vtable call per allocation) |
| **Performance** | Slightly faster (no virtual dispatch) | ~5-10ns overhead per allocation (virtual call) |
| **Composability** | Allocator must be copied/propagated manually | Resources are pointers — trivially shared |
| **Swap** | Only if allocators compare equal (or propagate_on_swap) | Only if resources are the same |
| **Stateful allocators** | Complex propagation traits | Simple — just pass a pointer |

```cpp

#include <memory_resource>
#include <vector>
#include <string>

// === Classic allocator problem ===
template <typename T>
struct StackAllocator { /* ... */ };

// These are DIFFERENT types:
using vec_default = std::vector<int>;
using vec_stack   = std::vector<int, StackAllocator<int>>;

// This function only works with one allocator type:
// void process(vec_default& v);     // can't pass vec_stack
// void process(vec_stack& v);       // can't pass vec_default

// === PMR solution ===
void process(std::pmr::vector<int>& v) {
    // Works regardless of which memory resource backs the vector
    v.push_back(42);
}

int main() {
    std::byte buf[4096];
    std::pmr::monotonic_buffer_resource arena(buf, sizeof(buf));

    std::pmr::vector<int> v1;            // default (new/delete)
    std::pmr::vector<int> v2(&arena);    // stack arena

    process(v1);  // OK
    process(v2);  // OK — same type!

    return 0;
}

```

**Key trade-off:** PMR gives you type erasure at the cost of one virtual function call per allocation. For most applications, this overhead is negligible. For ultra-hot inner loops with millions of tiny allocations, consider classic allocators.

### Q3: Show how pmr containers propagate the allocator to nested pmr containers

```cpp

#include <iostream>
#include <memory_resource>
#include <vector>
#include <map>
#include <string>

// === Custom tracking resource to see all allocations ===
class VerboseResource : public std::pmr::memory_resource {
    std::string name_;
    std::pmr::memory_resource* upstream_;
protected:
    void* do_allocate(size_t bytes, size_t align) override {
        std::cout << "[" << name_ << "] allocate " << bytes << " bytes\n";
        return upstream_->allocate(bytes, align);
    }
    void do_deallocate(void* p, size_t bytes, size_t align) override {
        upstream_->deallocate(p, bytes, align);
    }
    bool do_is_equal(const memory_resource& other) const noexcept override {
        return this == &other;
    }
public:
    VerboseResource(std::string name, std::pmr::memory_resource* upstream = std::pmr::get_default_resource())
        : name_(std::move(name)), upstream_(upstream) {}
};

int main() {
    VerboseResource mr("arena");

    // === Nested pmr containers: vector of vectors ===
    // The outer vector passes its allocator to inner vectors!
    std::pmr::vector<std::pmr::vector<int>> matrix(&mr);

    matrix.push_back(std::pmr::vector<int>{1, 2, 3});
    matrix.push_back(std::pmr::vector<int>{4, 5});

    // ALL allocations go through "arena":
    // [arena] allocate ... (outer vector storage)
    // [arena] allocate ... (inner vector [1,2,3])
    // [arena] allocate ... (inner vector [4,5])

    // === Map of strings (all pmr) ===
    std::pmr::map<std::pmr::string, std::pmr::vector<int>> data(&mr);
    data["scores"].push_back(95);
    data["scores"].push_back(87);
    data["names"];  // creates empty vector — still uses arena
    // [arena] allocate ... (map node)
    // [arena] allocate ... (string "scores")
    // [arena] allocate ... (vector storage)

    std::cout << "\ndata[\"scores\"]: ";
    for (int s : data["scores"]) std::cout << s << " ";
    std::cout << "\n";
    // Output: data["scores"]: 95 87

    // === How propagation works ===
    // pmr::polymorphic_allocator has uses_allocator construction support.
    // When a pmr container constructs a sub-element:
    //   1. It detects if the element type supports allocators (uses_allocator_v)
    //   2. If yes, it passes its own allocator to the element's constructor
    //   3. This happens automatically — no user code needed
    //
    // Requirement: nested containers must ALSO be pmr types!
    //   pmr::vector<pmr::vector<int>>  ✅ propagates
    //   pmr::vector<std::vector<int>>  ❌ inner vector ignores the allocator

    return 0;
}

```

**How it works:**

- PMR containers use `std::uses_allocator_v` to detect if their element type accepts an allocator.
- All `std::pmr::*` types accept a `polymorphic_allocator`, so propagation is automatic.
- The key requirement: **both** the outer and inner containers must be `pmr` types. `pmr::vector<std::vector<int>>` does NOT propagate — the inner `std::vector` ignores the allocator.
- This enables a single memory resource to back an entire hierarchy of containers, making arena and pool allocation practical for complex data structures.

---

## Notes

- **`get_default_resource()` / `set_default_resource()`** — controls the global default memory resource for all pmr containers that don't specify one. Default is `new_delete_resource()`.
- **monotonic_buffer_resource** is ideal for parse-then-discard patterns, frame allocators in games, and request-scoped allocations in servers.
- **pool_resource** (`unsynchronized_pool_resource`, `synchronized_pool_resource`) maintains free-lists of fixed-size blocks — good for frequent alloc/dealloc of same-sized objects.
- **`null_memory_resource()`** always throws `std::bad_alloc` — useful as an upstream to guarantee no fallback allocation occurs.
- **Thread safety:** `monotonic_buffer_resource` and `unsynchronized_pool_resource` are NOT thread-safe. Use `synchronized_pool_resource` for multi-threaded contexts.
- **C++20 addition:** `std::pmr::polymorphic_allocator` gained `allocate_bytes()`, `allocate_object<T>()`, `new_object<T>()` — making it easier to use outside of containers.
