# Design exception-safe vs noexcept library boundaries

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#e-error-handling>  

---

## Topic Overview

Library boundaries are where error handling strategy matters most. The choice between exceptions and noexcept affects performance, usability, and composability.

### Design Decision Matrix

| Scenario | Recommendation |
| --- | --- |
| Performance-critical inner loop | noexcept + error codes/expected |
| Constructor failure | Exceptions (constructors can't return error codes) |
| Destructor | Always noexcept (implicit since C++11) |
| Move operations | noexcept (required for `vector::push_back` optimization) |
| Swap | noexcept |
| C interop boundary | noexcept + error return |
| General library API | Exceptions or `std::expected` |

### The Dual-API Pattern

```cpp

#include <expected>
#include <stdexcept>
#include <string>
#include <fstream>
#include <system_error>

namespace mylib {

// Throwing version (convenient for scripts, applications)
std::string read_file(const std::string& path) {
    std::ifstream f(path);
    if (!f) throw std::runtime_error("Cannot open: " + path);
    return std::string(std::istreambuf_iterator<char>(f), {});
}

// Non-throwing version (for performance-critical code, embedded)
std::expected<std::string, std::error_code>
try_read_file(const std::string& path) noexcept {
    try {
        std::ifstream f(path);
        if (!f) return std::unexpected(
            std::make_error_code(std::errc::no_such_file_or_directory));
        return std::string(std::istreambuf_iterator<char>(f), {});
    } catch (...) {
        return std::unexpected(
            std::make_error_code(std::errc::io_error));
    }
}

} // namespace mylib

```

### Exception Safety at Library Boundaries

```cpp

// When your library throws, catch at the boundary and convert
// to a stable ABI-safe error type

#ifdef _WIN32
#define MYLIB_EXPORT __declspec(dllexport)
#else
#define MYLIB_EXPORT __attribute__((visibility("default")))
#endif

// C-compatible error return for shared library boundary
extern "C" {
    enum MyLibError { MYLIB_OK = 0, MYLIB_BAD_INPUT = 1, MYLIB_IO_ERROR = 2 };

    MYLIB_EXPORT MyLibError mylib_process(const char* input, char* output, size_t outlen) {
        try {
            // Internal C++ code can use exceptions freely
            std::string result = mylib::process(input);
            if (result.size() >= outlen) return MYLIB_BAD_INPUT;
            std::copy(result.begin(), result.end(), output);
            output[result.size()] = '\0';
            return MYLIB_OK;
        } catch (const std::invalid_argument&) {
            return MYLIB_BAD_INPUT;
        } catch (...) {
            return MYLIB_IO_ERROR;
        }
    }
}

```

---

## Self-Assessment

### Q1: Why must move constructors be noexcept for vector compatibility

`std::vector::push_back` uses `std::move_if_noexcept`. If the move constructor is not `noexcept`, vector falls back to copying during reallocation (to maintain the strong exception guarantee). A throwing move that fails mid-reallocation would leave elements in a moved-from state with no way to restore them.

### Q2: Design an error code enum that works with std::error_code

```cpp

#include <system_error>

enum class LibError {
    Success = 0,
    FileNotFound = 1,
    ParseError = 2,
    Timeout = 3,
};

struct LibErrorCategory : std::error_category {
    const char* name() const noexcept override { return "mylib"; }
    std::string message(int ev) const override {
        switch (static_cast<LibError>(ev)) {
            case LibError::Success: return "success";
            case LibError::FileNotFound: return "file not found";
            case LibError::ParseError: return "parse error";
            case LibError::Timeout: return "timeout";
            default: return "unknown error";
        }
    }
};

const LibErrorCategory& lib_error_category() {
    static LibErrorCategory instance;
    return instance;
}

std::error_code make_error_code(LibError e) {
    return {static_cast<int>(e), lib_error_category()};
}

namespace std {
    template<> struct is_error_code_enum<LibError> : true_type {};
}

```

### Q3: Explain the "catch everything at the boundary" pattern

At shared library boundaries (DLL/SO), exceptions must not escape. Different compilers/runtimes have incompatible exception ABIs. The boundary function catches all C++ exceptions inside a `try/catch(...)` block and converts them to error codes. Internally, the library can use exceptions freely.

---

## Notes

- **Exceptions cannot cross `extern "C"` boundaries** — always catch and convert.
- Move and swap should always be `noexcept` — this is the single most important `noexcept` decision.
- The dual-API pattern (throwing + `expected`) serves both convenience and performance users.
- `std::expected` (C++23) is the modern alternative to error codes at library boundaries.
