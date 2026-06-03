# Design error domains for large-scale multi-library projects

**Category:** Error Handling  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/error/error_code>  

---

## Topic Overview

In large projects with multiple libraries, each library defines its own error types. The question is: how do you make those types talk to each other without creating a tangled web of casts and string comparisons? `std::error_code` and `std::error_category` solve this by giving every library a standardized slot to plug into. Think of it as a universal adapter for errors: each library defines its own vocabulary (the enum), registers it with the standard system (the category), and from that point on all error codes in the entire program are interoperable through `std::error_code`.

### Creating an Error Domain

The pattern has three parts: an enum of codes, a category class that knows how to name and describe those codes, and a free function that combines them. Here's what a database library's error domain looks like in practice.

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

The `is_error_code_enum` specialization is what makes the implicit conversion work. Without it you'd need to call `make_error_code` by hand every time. With it, you can assign a `db::Error` directly to a `std::error_code` variable and the compiler handles the rest.

### Cross-Library Error Propagation

The real payoff is that once every library registers its own domain this way, a function can return errors from any of them through the same `std::error_code` type. The calling code can then inspect which domain the error came from and act accordingly.

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

Notice that `process_request` doesn't need to know anything about the internal structure of the database or network error types. It just passes them through as `std::error_code`. The caller then asks the category for its identity and dispatches accordingly. This is why the category singleton must have a stable, unique address - equality is checked by address, not by name string.

---

## Self-Assessment

### Q1: What's the difference between error_code and error_condition

`error_code` is a specific error from a specific domain (e.g., `ENOENT` from POSIX). `error_condition` is a portable, abstract error (e.g., `file_not_found`). An `error_code` can be compared to an `error_condition` for portable error checking across different platforms or domains that all mean "file not found" even if their raw integer values differ.

### Q2: How do error domains compose

Each library defines its own `error_category` with a unique name. All error codes are interoperable via `std::error_code`, which stores {int, category*}. Libraries can translate between domains or propagate foreign errors unchanged because the category pointer identifies whose error it is regardless of what the integer value means.

### Q3: When to use error_code vs exceptions vs expected

Use error codes for C ABI boundaries and library interfaces where exceptions cannot safely cross. Use exceptions for truly exceptional conditions that are rare and where the automatic propagation through the call stack is valuable. Use `std::expected` for modern APIs where failure is common enough that the caller should always handle it explicitly. All three can coexist in the same project - the trick is to pick the right tool for each layer.

---

## Notes

- `std::error_code` is the standard interop format for errors between C++ libraries.
- Each category singleton must have a unique address - use Meyer's singleton (a function with a local static) to guarantee this.
- `std::error_condition` enables portable error comparison across platforms even when the underlying integer codes differ.
- Boost.System introduced this pattern; it was adopted into the C++ standard in C++11.
