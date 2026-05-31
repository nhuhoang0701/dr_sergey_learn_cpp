# Use std::make_shared_for_overwrite and std::make_unique_for_overwrite (C++20)

**Category:** Memory and Ownership  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/memory/shared_ptr/make_shared>  

---

## Topic Overview

`make_shared` and `make_unique` value-initialize (zero-fill) the allocated memory. For large buffers that will be immediately overwritten, this is wasteful. C++20 adds `_for_overwrite` variants that default-initialize instead, leaving memory uninitialized for trivial types.

### Comparison

Here is the simplest view of what changes between the two families:

```cpp
#include <memory>
#include <iostream>

int main() {
    // Value-initialized: all bytes set to 0
    auto buf1 = std::make_unique<int[]>(1'000'000);
    // buf1[0] == 0, buf1[999999] == 0 (guaranteed)

    // Default-initialized: bytes are indeterminate (for int)
    auto buf2 = std::make_unique_for_overwrite<int[]>(1'000'000);
    // buf2[0] == ??? (uninitialized - will be overwritten anyway)

    // Same for shared_ptr:
    auto sbuf1 = std::make_shared<double[]>(1'000'000);              // Zero-filled
    auto sbuf2 = std::make_shared_for_overwrite<double[]>(1'000'000); // Uninitialized
}
```

### When It Matters

The savings are most visible when the buffer is large and the contents will be immediately replaced by an I/O read, a `memcpy`, or similar:

```cpp
#include <memory>
#include <fstream>
#include <cstring>

void read_file_optimized(const std::string& path) {
    auto size = std::filesystem::file_size(path);

    // BAD: zeros 10MB then immediately overwrites with file contents
    auto buf = std::make_unique<char[]>(size);
    // memset(buf.get(), 0, size) happens implicitly - wasted!

    // GOOD: skip zeroing - the read() will fill the buffer
    auto buf2 = std::make_unique_for_overwrite<char[]>(size);

    std::ifstream f(path, std::ios::binary);
    f.read(buf2.get(), size);
    // Buffer was never uselessly zeroed
}

// Performance difference for 1GB buffer:
// make_unique<char[]>(1GB):               ~250ms (memset 1GB)
// make_unique_for_overwrite<char[]>(1GB):  ~0ms (no initialization)
```

---

## Self-Assessment

### Q1: What types benefit from `_for_overwrite`

Trivial types (`int`, `double`, `char`, POD structs). For non-trivial types, default-initialization calls the default constructor anyway, so there's no savings. The main use case is large arrays of scalars.

### Q2: Is reading from a `_for_overwrite` buffer before writing UB

For `int` and other non-class types: yes, reading an indeterminate value is UB (C++20) or erroneous behavior (C++26). For class types with default constructors: the constructor runs, so objects are properly initialized.

### Q3: Show the performance impact

```cpp
Benchmark: allocate 100MB buffer
make_unique<char[]>(100MB):               45ms  (zeros memory)
make_unique_for_overwrite<char[]>(100MB):  0.1ms (no zeroing)
Speedup: ~450x for allocation alone
```

The speedup scales with buffer size. On a 1 GB buffer it easily adds up to hundreds of milliseconds saved just on initialization that was going to be thrown away anyway.

---

## Notes

- Use `_for_overwrite` when the buffer will be immediately filled (I/O, DMA, memcpy).
- Do NOT read from the buffer before writing - use ASan/MSan to catch violations.
- The `_for_overwrite` variants exist for both `unique_ptr` and `shared_ptr`, arrays and single objects.
- This is the smart pointer equivalent of `new int` (default-init) vs `new int()` (value-init).
