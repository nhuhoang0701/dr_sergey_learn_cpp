# Understand `std::error_condition` vs `std::error_code` and When to Use Each

**Category:** Error Handling  
**Standard:** C++11  
**Reference:** [cppreference - error_code](https://en.cppreference.com/w/cpp/error/error_code), [cppreference - error_condition](https://en.cppreference.com/w/cpp/error/error_condition)  

---

## Topic Overview

The `<system_error>` header provides two complementary types for describing errors. They look similar but serve different purposes, and confusing them leads to brittle, non-portable error checking. The distinction is worth understanding clearly.

| Aspect | `std::error_code` | `std::error_condition` |
| --- | --- | --- |
| **Semantics** | Platform-specific, concrete error | Portable, abstract error condition |
| **Typical source** | OS call, library function | Comparison target in application logic |
| **Category example** | `std::system_category()` (POSIX `errno` / Win32) | `std::generic_category()` (portable POSIX-like) |
| **Who creates it?** | Low-level code, OS wrappers | Application or library authors for matching |
| **Compared how?** | `==` against an `error_condition` via category mapping | `==` against an `error_code` - delegated to category |

The design follows a bridge pattern. An `error_code` carries the raw integer value *and* a reference to its `error_category`. When you compare `error_code == error_condition`, the category's virtual `equivalent()` function decides if they match, even if the numeric values differ across platforms. This is what lets Linux `errno` 2 and Windows `ERROR_FILE_NOT_FOUND` both compare equal to the portable `std::errc::no_such_file_or_directory`.

Here is a picture of how the types relate:

```cpp
//         error_code       equivalent()?       error_condition
//         value: 13   <---------------------->  value: EACCES
//         cat: system                           cat: generic
//              |                                      |
//              v                                      v
//      system_category  -- default_error_   generic_category
//      (Win32 / POSIX)     condition() -->  (portable POSIX)
```

The key rule: **write `error_code` when you report an error, use `error_condition` when you check for an error.** Library code returns `error_code` so the caller gets the exact platform error. Application code compares against `std::errc` enumerators (which implicitly convert to `error_condition`) for portable branching.

A custom `error_category` overrides `default_error_condition()` to map platform-specific integer codes to generic conditions, and overrides `equivalent()` to support cross-category comparison. This is how Windows `ERROR_ACCESS_DENIED (5)` maps to the portable `std::errc::permission_denied`.

---

## Self-Assessment

### Q1: Show how a platform `error_code` compares equal to a portable `error_condition`

The comparison works even though the two objects come from different categories with different integer values. That is the whole point of the `equivalent()` mechanism. Watch how the same filesystem error produces matching comparisons on both Linux and Windows through the same portable check:

```cpp
// error_equivalence.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o error_equivalence error_equivalence.cpp
#include <cerrno>
#include <iostream>
#include <system_error>
#include <fstream>

int main() {
    // Attempt to open a non-existent file to get a real OS error
    std::ifstream f("/no/such/file/ever");
    if (!f) {
        // error_code from errno - platform-specific
        std::error_code ec(errno, std::system_category());

        // error_condition - portable comparison target
        std::error_condition not_found = std::errc::no_such_file_or_directory;

        std::cout << "error_code:      " << ec.category().name()
                  << " / " << ec.value() << " / " << ec.message() << "\n";
        std::cout << "error_condition:  " << not_found.category().name()
                  << " / " << not_found.value() << " / " << not_found.message() << "\n";

        // Cross-category comparison works through equivalent()
        if (ec == std::errc::no_such_file_or_directory) {
            std::cout << "Match: file not found (portable check)\n";
        }
        if (ec == not_found) {
            std::cout << "Match: error_code == error_condition\n";
        }

        // Direct integer comparison would be fragile - don't do this:
        // if (ec.value() == ENOENT)  // non-portable on Windows
    }
}
// Typical output (Linux):
// error_code:      system / 2 / No such file or directory
// error_condition:  generic / 2 / No such file or directory
// Match: file not found (portable check)
// Match: error_code == error_condition
```

The two values happen to be the same integer on Linux, but you should never rely on that. Use the portable `std::errc` comparison every time.

### Q2: Create a custom `error_category` that maps domain-specific codes to generic conditions

When you write a library with its own error codes - say, a database layer - you want callers to be able to write portable checks like `ec == std::errc::connection_refused` even though your error code is `DbError::connection_refused`. The `default_error_condition()` override in your category is what makes that mapping happen:

```cpp
// custom_category.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o custom_category custom_category.cpp
#include <iostream>
#include <string>
#include <system_error>

// --- Domain-specific error codes for a database layer ---
enum class DbError {
    ok                = 0,
    connection_refused = 1,
    auth_failed        = 2,
    query_timeout      = 3,
    table_not_found    = 4,
};

class DbCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "database"; }

    std::string message(int ev) const override {
        switch (static_cast<DbError>(ev)) {
            case DbError::ok:                return "success";
            case DbError::connection_refused: return "database connection refused";
            case DbError::auth_failed:        return "authentication failed";
            case DbError::query_timeout:      return "query timed out";
            case DbError::table_not_found:    return "table not found";
        }
        return "unknown database error";
    }

    // Map domain codes to portable generic conditions
    std::error_condition default_error_condition(int ev) const noexcept override {
        switch (static_cast<DbError>(ev)) {
            case DbError::connection_refused:
                return std::errc::connection_refused;
            case DbError::auth_failed:
                return std::errc::permission_denied;
            case DbError::table_not_found:
                return std::errc::no_such_file_or_directory; // closest portable match
            default:
                return std::error_condition(ev, *this);
        }
    }
};

const DbCategory& db_category() {
    static DbCategory instance;
    return instance;
}

std::error_code make_error_code(DbError e) {
    return {static_cast<int>(e), db_category()};
}

// Register with the type system
namespace std {
    template <> struct is_error_code_enum<DbError> : true_type {};
}

int main() {
    std::error_code ec = DbError::connection_refused;

    std::cout << "code:  " << ec.category().name()
              << " / " << ec.value() << " - " << ec.message() << "\n";

    // Portable comparison - works because default_error_condition maps it
    if (ec == std::errc::connection_refused) {
        std::cout << "Portable match: connection_refused\n";
    }

    ec = DbError::auth_failed;
    if (ec == std::errc::permission_denied) {
        std::cout << "Portable match: permission_denied\n";
    }

    ec = DbError::query_timeout;
    // No generic mapping - falls back to domain-specific condition
    if (ec != std::errc::timed_out) {
        std::cout << "query_timeout has no generic mapping to timed_out\n";
        std::cout << "condition: " << ec.default_error_condition().category().name()
                  << " / " << ec.default_error_condition().value() << "\n";
    }
}
// Output:
// code:  database / 1 - database connection refused
// Portable match: connection_refused
// Portable match: permission_denied
// query_timeout has no generic mapping to timed_out
// condition: database / 3
```

Notice that `query_timeout` has no natural mapping to any standard `errc` value. In that case `default_error_condition` falls back to returning the code as its own condition within the database category. Callers who need to handle that case must compare against `DbError::query_timeout` directly.

### Q3: Demonstrate why you should never compare `error_code::value()` directly across categories

This is the classic trap. Two completely different error domains can independently assign the integer value `1` to completely different meanings. If you compare raw integers, you get false positives. The `error_code` `operator==` guards against this by checking both the value and the category:

```cpp
// value_trap.cpp - C++17
// Compile: g++ -std=c++17 -O2 -Wall -Wextra -o value_trap value_trap.cpp
#include <cerrno>
#include <iostream>
#include <system_error>

// Two error domains that happen to use the same integer value
enum class NetError  { timeout = 1, dns_failure = 2 };
enum class FileError { not_found = 1, read_only = 2 };

class NetErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "network"; }
    std::string message(int ev) const override {
        switch (static_cast<NetError>(ev)) {
            case NetError::timeout:     return "network timeout";
            case NetError::dns_failure: return "DNS resolution failed";
        }
        return "unknown network error";
    }
};

class FileErrorCategory : public std::error_category {
public:
    const char* name() const noexcept override { return "filesystem"; }
    std::string message(int ev) const override {
        switch (static_cast<FileError>(ev)) {
            case FileError::not_found: return "file not found";
            case FileError::read_only: return "file is read-only";
        }
        return "unknown file error";
    }
};

const NetErrorCategory&  net_cat()  { static NetErrorCategory  c; return c; }
const FileErrorCategory& file_cat() { static FileErrorCategory c; return c; }

std::error_code make_error_code(NetError e)  { return {static_cast<int>(e), net_cat()}; }
std::error_code make_error_code(FileError e) { return {static_cast<int>(e), file_cat()}; }

namespace std {
    template <> struct is_error_code_enum<NetError>  : true_type {};
    template <> struct is_error_code_enum<FileError> : true_type {};
}

int main() {
    std::error_code net_err  = NetError::timeout;    // value = 1
    std::error_code file_err = FileError::not_found;  // value = 1

    // BAD: raw integer comparison gives a false positive
    if (net_err.value() == file_err.value()) {
        std::cout << "[BUG] Raw values match: "
                  << net_err.value() << " == " << file_err.value() << "\n";
    }

    // GOOD: operator== checks BOTH value AND category
    if (net_err != file_err) {
        std::cout << "[OK]  error_code comparison correctly distinguishes:\n"
                  << "  " << net_err.category().name()  << ":" << net_err.value()
                  << " != " << file_err.category().name() << ":" << file_err.value() << "\n";
    }

    // GOOD: compare against a specific enum
    if (net_err == NetError::timeout) {
        std::cout << "[OK]  Correctly identified as network timeout\n";
    }
}
// Output:
// [BUG] Raw values match: 1 == 1
// [OK]  error_code comparison correctly distinguishes:
//   network:1 != filesystem:1
// [OK]  Correctly identified as network timeout
```

The output drives the point home: raw integer `1 == 1` is true (a false positive), but `error_code == error_code` correctly returns false because the categories differ. Always use the full `error_code` comparison.

---

## Notes

- **`error_code`** = "what happened" (platform detail). **`error_condition`** = "what kind of problem is this?" (portable concept).
- `std::errc` enumerators map to `error_condition` via `is_error_condition_enum`. Use them as portable comparison targets.
- Never compare `.value()` integers unless you also compare `.category()`. Two categories can (and do) reuse the same integer.
- `default_error_condition()` is the category's opportunity to translate a platform-specific code into a generic condition. Override it in custom categories.
- `equivalent()` is virtual with two overloads: one on `error_category` for `error_code` vs `int+condition`, another for `int+code` vs `error_condition`. This powers the `==` operator.
- `system_category()` reflects the OS error space (`errno` on POSIX, `GetLastError()` on Windows). `generic_category()` reflects the portable POSIX subset.
- When designing library APIs, return `error_code` from functions and let callers compare against `error_condition` / `std::errc` - this keeps the API portable while preserving platform detail.
