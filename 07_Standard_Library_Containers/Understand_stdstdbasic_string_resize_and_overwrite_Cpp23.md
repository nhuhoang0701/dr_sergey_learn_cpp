# Understand std::basic_string::resize_and_overwrite (C++23)

**Category:** Standard Library Containers  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string/resize_and_overwrite>  

---

## Topic Overview

`resize_and_overwrite` grows a string and lets you fill the buffer directly without double-initialization. The callable receives a raw pointer and the buffer size, and returns the actual used length.

### API

```cpp

#include <string>
#include <cstring>
#include <iostream>

int main() {
    std::string s;

    // resize_and_overwrite(count, operation)
    // operation receives (char* buf, size_t buf_size) and returns actual size
    s.resize_and_overwrite(100, [](char* buf, size_t n) -> size_t {
        // buf has space for n chars (uninitialized!)
        std::strcpy(buf, "Hello, World!");
        return 13;  // Actual length used
    });

    std::cout << s << "\n";         // "Hello, World!"
    std::cout << s.size() << "\n";  // 13
}

```

### Real-World: snprintf Without Double-Init

```cpp

#include <string>
#include <cstdio>

std::string format_number(double value) {
    std::string result;
    result.resize_and_overwrite(64, [value](char* buf, size_t n) -> size_t {
        int written = std::snprintf(buf, n + 1, "%.6f", value);
        return written > 0 ? static_cast<size_t>(written) : 0;
    });
    return result;
}

```

### Performance Comparison

```cpp

#include <string>
#include <cstring>
#include <chrono>
#include <iostream>

// Old way: resize + copy (double-init)
std::string old_way(const char* src, size_t len) {
    std::string s;
    s.resize(len);              // Zero-initializes len bytes
    std::memcpy(s.data(), src, len);  // Overwrites — wasted zero-init
    return s;
}

// New way: resize_and_overwrite (single init)
std::string new_way(const char* src, size_t len) {
    std::string s;
    s.resize_and_overwrite(len, [src, len](char* buf, size_t) -> size_t {
        std::memcpy(buf, src, len);  // Direct fill — no double-init
        return len;
    });
    return s;
}

```

---

## Self-Assessment

### Q1: What are the rules for the callable passed to resize_and_overwrite

The callable receives `(charT* buf, size_t count)` where `buf` points to allocated (but uninitialized) storage of at least `count` chars. It must return a value `<= count` indicating the actual used size. It must not throw. Accessing `buf[count]` or beyond is UB.

### Q2: Can you shrink the string with resize_and_overwrite

Yes. The callable can return any value from 0 to count. The string's final size equals the return value. This is useful when the actual content size isn't known until the write operation completes.

### Q3: How does this help with C API interop

```cpp

std::string get_error_message(int errcode) {
    std::string s;
    s.resize_and_overwrite(256, [errcode](char* buf, size_t n) -> size_t {
        // C API writes into buffer, returns length
        return strerror_r(errcode, buf, n);
    });
    return s;
}

```

---

## Notes

- Available in GCC 12+, Clang 14+, MSVC 19.31+.
- The callable must not access the string object itself (undefined behavior).
- The callable receives `n` where `n >= count` requested (implementation may over-allocate).
- For `std::wstring`, the callable receives `wchar_t*`.
