# Know std::vector resize-and-overwrite-style patterns for avoiding double-initialization

**Category:** Standard Library Containers  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string/resize_and_overwrite>  

---

## Topic Overview

`vector::resize(n)` and `string::resize(n)` value-initialize new elements. If you immediately overwrite them, the initialization was wasted. C++23 provides `resize_and_overwrite` for strings; for vectors, patterns exist to avoid double-init.

### The Problem

```cpp

#include <vector>
#include <string>

void read_into_vector(int fd) {
    std::vector<char> buf;
    buf.resize(4096);       // Zero-initializes 4096 bytes
    auto n = read(fd, buf.data(), 4096);  // Immediately overwrites them
    buf.resize(n);
    // The initial zeroing was completely wasted
}

```

### C++23: string::resize_and_overwrite

```cpp

#include <string>

void fill_string(int fd) {
    std::string s;
    s.resize_and_overwrite(4096, [fd](char* buf, size_t count) -> size_t {
        // buf is uninitialized memory!
        auto n = read(fd, buf, count);
        return n > 0 ? n : 0;  // Return actual size
    });
    // No double-initialization: buffer is filled directly
}

```

### Vector Patterns

```cpp

#include <vector>
#include <memory>
#include <cstring>

// Pattern 1: Reserve + placement
template<typename T>
void build_vector_no_init(std::vector<T>& v, size_t count) {
    v.reserve(count);
    // Use emplace_back to construct elements directly:
    for (size_t i = 0; i < count; ++i)
        v.emplace_back(compute_value(i));  // No default + assign, just construct
}

// Pattern 2: Use unique_ptr<T[]> for raw buffers
auto buf = std::make_unique_for_overwrite<char[]>(4096);
auto n = read(fd, buf.get(), 4096);  // Zero wasted initialization

// Pattern 3: Propose resize_and_overwrite for vector (P1072)
// Not yet standardized but under discussion

```

---

## Self-Assessment

### Q1: When does double-initialization matter

For large buffers (MB+) in I/O operations, network receive buffers, and hot loops that allocate/fill. For small vectors (<1KB), the overhead is negligible. Profile before optimizing.

### Q2: Why doesn't vector have resize_and_overwrite

`string::resize_and_overwrite` works because string elements are `char` (trivially constructible). For `vector<T>`, elements may have non-trivial constructors — the standard can't skip construction safely. P1072 proposes `resize_default_init` for trivial types.

### Q3: What is the performance difference

```cpp

Benchmark: resize 1GB vector<char>
resize(1GB):                     ~250ms (memset to 0)
reserve + read(fd, buf, 1GB):   ~0ms for allocation (zero-init skipped)
Speedup: significant for I/O-bound code

```

---

## Notes

- `string::resize_and_overwrite` is C++23's solution for string buffers.
- For vectors, prefer `reserve()` + `push_back/emplace_back` over `resize()` + overwrite.
- `make_unique_for_overwrite<T[]>(n)` (C++20) is the cleanest way to get an uninitialized buffer.
- P1072 (`basic_string::resize_default_init`) was merged into `resize_and_overwrite`.
