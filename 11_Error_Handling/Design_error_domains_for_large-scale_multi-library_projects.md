# Design error domains for large-scale multi-library projects

**Category:** Error Handling  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/error/error_code>  

---

## Topic Overview

In large projects with multiple libraries, each library defines its own error types. `std::error_code` and `std::error_category` provide a standardized way to create interoperable error domains.

### Creating an Error Domain

```cpp

#include <system_error>
#include <string>

// Library A: database errors
namespace db {
    enum class Error {
        Success = 0,
        ConnectionFailed = 1,
        QueryFailed = 2,
        Timeout = 3,
        AuthFailed = 4,
    };

    struct ErrorCategory : std::error_category {
        const char* name() const noexcept override { return "database"; }
        std::string message(int ev) const override {
            switch (static_cast<Error>(ev)) {
                case Error::Success: return "success";
                case Error::ConnectionFailed: return "connection failed";
                case Error::QueryFailed: return "query failed";
                case Error::Timeout: return "timeout";
                case Error::AuthFailed: return "authentication failed";
                default: return "unknown database error";
            }
        }
    };

    const ErrorCategory& error_category() {
        static ErrorCategory instance;
        return instance;
    }

    std::error_code make_error_code(Error e) {
        return {static_cast<int>(e), error_category()};
    }
}

namespace std {
    template<> struct is_error_code_enum<db::Error> : true_type {};
}

// Now db::Error implicitly converts to std::error_code:
std::error_code ec = db::Error::Timeout;
std::cout << ec.category().name() << ": " << ec.message() << "\n";
// Output: "database: timeout"

```

### Cross-Library Error Propagation

```cpp

#include <system_error>

// Library B can use any error domain:
std::error_code process_request() {
    auto db_result = db::connect();
    if (db_result)
        return db_result;  // db::Error propagated as std::error_code

    auto net_result = net::send_response();
    if (net_result)
        return net_result;  // net::Error propagated as std::error_code

    return {};  // Success
}

// The caller can check the domain:
auto ec = process_request();
if (ec.category() == db::error_category()) {
    // Database-specific recovery
} else if (ec.category() == net::error_category()) {
    // Network-specific recovery
}

```

---

## Self-Assessment

### Q1: What's the difference between error_code and error_condition

`error_code` is a specific error from a specific domain (e.g., `ENOENT` from POSIX). `error_condition` is a portable, abstract error (e.g., `file_not_found`). An error_code can be compared to an error_condition for portable error checking.

### Q2: How do error domains compose

Each library defines its own `error_category` with a unique name. All error codes are interoperable via `std::error_code`, which stores {int, category*}. Libraries can translate between domains or propagate foreign errors unchanged.

### Q3: When to use error_code vs exceptions vs expected

Error codes for C ABI boundaries and library interfaces. Exceptions for truly exceptional conditions. `std::expected` for modern APIs where the caller should handle the error. All three can coexist in the same project.

---

## Notes

- `std::error_code` is the standard interop format for errors between C++ libraries.
- Each category singleton must have a unique address — use Meyer's singleton.
- `std::error_condition` enables portable error comparison across platforms.
- Boost.System introduced this pattern; it was adopted into the C++ standard.
