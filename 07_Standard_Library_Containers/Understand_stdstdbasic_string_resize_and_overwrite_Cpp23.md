# Understand std::basic_string::resize_and_overwrite (C++23)

**Category:** Standard Library Containers  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string/resize_and_overwrite>  

---

## Topic Overview

`resize_and_overwrite` grows a string and lets you fill the buffer directly without double-initialization. The callable receives a raw pointer and the buffer size, and returns the actual used length.

Here is the problem it solves. The old way to get data into a `std::string` from a C API or a low-level write operation was: call `s.resize(n)` to allocate space (which zero-initializes every byte), then write your actual content on top of those zeros. That zero-initialization is wasted work - you are writing every byte twice. `resize_and_overwrite` lets the implementation hand you the uninitialized buffer directly, so you write each byte exactly once.

### API

The callable signature is `(charT* buf, size_t n) -> size_t`. You write into `buf`, then return however many characters you actually used. The string's final `size()` will equal your return value:

```cpp
#include <string>
#include <cstring>
#include <iostream>

int main() {
    std::string s;

    // resize_and_overwrite(count, operation)
    // operation receives (char* buf, size_t n) and returns actual size
    s.resize_and_overwrite(100, [](char* buf, size_t n) -> size_t {
        // buf has space for n chars (uninitialized!)
        std::strcpy(buf, "Hello, World!");
        return 13;  // Actual length used
    });

    std::cout << s << "\n";         // "Hello, World!"
    std::cout << s.size() << "\n";  // 13
}
```

The buffer has room for `n` characters but you only used 13, so the string ends up with `size() == 13`. No double-init, no separate resize step.

### Real-World: snprintf Without Double-Init

A very practical use case is wrapping `snprintf`. Normally you would `resize` then `snprintf`, writing the null terminator zone twice. Here you skip straight to the write:

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

Notice `n + 1` in the `snprintf` call: the implementation may allocate slightly more than `count` bytes (it rounds up to the nearest internal allocation unit), so `n` is the actual usable space, and you pass `n + 1` to allow `snprintf` its null terminator without reading into invalid memory.

### Performance Comparison

The difference between the old pattern and the new one is a single extra `memset` worth of work per call. For small strings that is noise; for large strings in a tight loop it matters:

```cpp
#include <string>
#include <cstring>
#include <chrono>
#include <iostream>

// Old way: resize + copy (double-init)
std::string old_way(const char* src, size_t len) {
    std::string s;
    s.resize(len);              // Zero-initializes len bytes
    std::memcpy(s.data(), src, len);  // Overwrites - wasted zero-init
    return s;
}

// New way: resize_and_overwrite (single init)
std::string new_way(const char* src, size_t len) {
    std::string s;
    s.resize_and_overwrite(len, [src, len](char* buf, size_t) -> size_t {
        std::memcpy(buf, src, len);  // Direct fill - no double-init
        return len;
    });
    return s;
}
```

For large buffers (kilobytes and above), `new_way` avoids zeroing that memory first, which can make a measurable difference when called repeatedly.

---

## Self-Assessment

### Q1: What are the rules for the callable passed to resize_and_overwrite

The callable receives `(charT* buf, size_t count)` where `buf` points to allocated (but uninitialized) storage of at least `count` chars. It must return a value `<= count` indicating the actual used size. It must not throw. Accessing `buf[count]` or beyond is undefined behavior - the buffer is exactly `count` characters wide from the callable's perspective (the implementation may have allocated more, but you cannot rely on that).

### Q2: Can you shrink the string with resize_and_overwrite

Yes. The callable can return any value from 0 to `count`. The string's final size equals the return value. This is useful when the actual content size is not known until the write operation completes - you request a generous upper bound, write what you can, and return the true length.

### Q3: How does this help with C API interop

C APIs that write into a caller-supplied buffer are exactly the pattern `resize_and_overwrite` was designed for. You allocate the buffer inside the string, pass it to the C function, and return whatever the C function reports as the written length:

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

No intermediate `std::vector<char>`, no manual size tracking, no double-initialization. The string owns the buffer from the start.

---

## Notes

- Available in GCC 12+, Clang 14+, MSVC 19.31+.
- The callable must not access the string object itself (undefined behavior).
- The callable receives `n` where `n >= count` requested (implementation may over-allocate).
- For `std::wstring`, the callable receives `wchar_t*`.
