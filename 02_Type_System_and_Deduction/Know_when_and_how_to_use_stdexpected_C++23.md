# Know when and how to use std::expected (C++23)

**Category:** Type System & Deduction  
**Item:** #24  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

`std::expected<T, E>` holds either a value of type `T` (success) or an error of type `E` (failure). It's a vocabulary type for error handling without exceptions.

### Basic Usage

```cpp

#include <expected>
#include <string>
#include <iostream>

std::expected<int, std::string> parse_int(const std::string& s) {
    try {
        return std::stoi(s);  // returns expected holding int
    } catch (...) {
        return std::unexpected("invalid integer: " + s);  // returns error
    }
}

void demo() {
    auto result = parse_int("42");
    if (result) {
        std::cout << "Value: " << *result << "\n";   // 42
    }

    auto bad = parse_int("abc");
    if (!bad) {
        std::cout << "Error: " << bad.error() << "\n";  // invalid integer: abc
    }
}

```

### Key API

| Method | Description |
| --- | --- |
| `has_value()` / `operator bool()` | Check if it holds a value |
| `value()` | Get value (throws `bad_expected_access` if error) |
| `*result` / `result->member` | Unchecked access to value |
| `error()` | Get the error |
| `value_or(default)` | Value if present, otherwise default |
| `and_then(f)` | Monadic: chain on success |
| `or_else(f)` | Monadic: chain on error |
| `transform(f)` | Monadic: map the value |
| `transform_error(f)` | Monadic: map the error |

### Monadic Operations (Chaining)

```cpp

auto result = parse_int(input)
    .and_then([](int n) -> std::expected<int, std::string> {
        if (n < 0) return std::unexpected("must be positive");
        return n;
    })
    .transform([](int n) { return n * 2; })
    .or_else([](std::string err) -> std::expected<int, std::string> {
        std::cerr << "Error: " << err << "\n";
        return 0;  // recover with default
    });

```

---

## Self-Assessment

### Q1: Rewrite a function that throws on error to return `std::expected<T, E>`

```cpp

#include <expected>
#include <string>
#include <iostream>
#include <fstream>
#include <sstream>

// BEFORE: throws on error
std::string read_file_throwing(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open())
        throw std::runtime_error("Cannot open file: " + path);

    std::stringstream ss;
    ss << file.rdbuf();
    if (file.bad())
        throw std::runtime_error("Read error: " + path);

    return ss.str();
}

// AFTER: returns expected — no exceptions!
enum class FileError { NotFound, ReadError, Empty };

std::expected<std::string, FileError> read_file_expected(const std::string& path) {
    std::ifstream file(path);
    if (!file.is_open())
        return std::unexpected(FileError::NotFound);

    std::stringstream ss;
    ss << file.rdbuf();
    if (file.bad())
        return std::unexpected(FileError::ReadError);

    std::string content = ss.str();
    if (content.empty())
        return std::unexpected(FileError::Empty);

    return content;  // implicit conversion to expected holding value
}

std::string error_to_string(FileError e) {
    switch (e) {
        case FileError::NotFound:  return "file not found";
        case FileError::ReadError: return "read error";
        case FileError::Empty:     return "file is empty";
    }
    return "unknown error";
}

int main() {
    auto result = read_file_expected("config.txt");

    if (result) {
        std::cout << "Content: " << result.value() << "\n";
    } else {
        std::cout << "Failed: " << error_to_string(result.error()) << "\n";
    }

    // value_or for defaults:
    std::string content = read_file_expected("missing.txt")
                            .value_or("default config");
    std::cout << "Config: " << content << "\n";
}

```

### Q2: Chain multiple `std::expected` operations using `and_then`

```cpp

#include <expected>
#include <string>
#include <iostream>
#include <charconv>

using Error = std::string;

std::expected<std::string, Error> get_input() {
    // Simulate getting user input
    return std::string("42");
}

std::expected<int, Error> to_int(const std::string& s) {
    int value;
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), value);
    if (ec != std::errc{})
        return std::unexpected("not a number: " + s);
    return value;
}

std::expected<int, Error> validate_range(int n) {
    if (n < 1 || n > 100)
        return std::unexpected("out of range: " + std::to_string(n));
    return n;
}

std::expected<double, Error> compute(int n) {
    if (n == 0) return std::unexpected("division by zero");
    return 100.0 / n;
}

int main() {
    // WITHOUT monadic chaining (nested if-checks):
    {
        auto input = get_input();
        if (!input) { std::cout << input.error(); return 1; }
        auto num = to_int(*input);
        if (!num) { std::cout << num.error(); return 1; }
        auto valid = validate_range(*num);
        if (!valid) { std::cout << valid.error(); return 1; }
        auto result = compute(*valid);
        if (!result) { std::cout << result.error(); return 1; }
        std::cout << "Result: " << *result << "\n";
    }

    // WITH monadic chaining — flat pipeline, no nesting:
    {
        auto result = get_input()
            .and_then(to_int)                    // string → int (or error)
            .and_then(validate_range)            // int → int (or error)
            .and_then(compute)                   // int → double (or error)
            .transform([](double d) {            // double → string
                return "Answer: " + std::to_string(d);
            })
            .or_else([](const Error& e) -> std::expected<std::string, Error> {
                return "Fallback (error was: " + e + ")";
            });

        std::cout << *result << "\n";
    }
}

```

### Q3: Compare `std::expected` vs `std::optional` for error-propagating pipelines

| Aspect | `std::optional<T>` | `std::expected<T, E>` |
| --- | --- | --- |
| **Error info** | None — just "no value" | Rich — carries error of type `E` |
| **Use case** | Value may or may not exist | Operation can fail with a reason |
| **Monadic ops** | `and_then`, `transform`, `or_else` (C++23) | Same set + `transform_error` |
| **Error reporting** | Cannot distinguish WHY it failed | Error type describes the failure |
| **Performance** | Slightly smaller (no error storage) | Slightly larger (stores error) |
| **Equivalent** | `expected<T, std::monostate>` ≈ `optional<T>` | — |

```cpp

#include <expected>
#include <optional>
#include <string>
#include <iostream>
#include <unordered_map>

// optional: good for "maybe" semantics
std::optional<int> find_user_id(const std::string& name) {
    static std::unordered_map<std::string, int> db = {{"Alice", 1}, {"Bob", 2}};
    if (auto it = db.find(name); it != db.end())
        return it->second;
    return std::nullopt;  // No information about WHY it wasn't found
}

// expected: good for error-reporting semantics
enum class LookupError { NotFound, DatabaseDown, PermissionDenied };

std::expected<int, LookupError> find_user_id_v2(const std::string& name) {
    // Simulate different failure modes
    if (name.empty())
        return std::unexpected(LookupError::PermissionDenied);
    if (name == "Alice")
        return 1;
    return std::unexpected(LookupError::NotFound);
}

int main() {
    // optional pipeline — can only say "it failed"
    auto id = find_user_id("Charlie");
    if (!id) std::cout << "Not found (but why?)\n";

    // expected pipeline — can explain WHY it failed
    auto id2 = find_user_id_v2("Charlie");
    if (!id2) {
        switch (id2.error()) {
            case LookupError::NotFound:         std::cout << "User not found\n"; break;
            case LookupError::DatabaseDown:      std::cout << "DB is down\n"; break;
            case LookupError::PermissionDenied:  std::cout << "Access denied\n"; break;
        }
    }
}

```

---

## Notes

- `std::unexpected(error)` is used to construct an expected in the error state — similar to how `std::nullopt` creates an empty optional.
- `expected<void, E>` is valid — useful for functions that have no return value but can fail.
- `and_then` must return `expected<U, E>` (same error type). Use `transform` to just map the value without changing expected.
- Prefer `expected` over error codes — it's harder to ignore (no implicit conversion to bool like `int`).
- `expected` does NOT replace exceptions for truly exceptional cases (out of memory, logic errors) — it's for expected failure paths.

**Illustration:**

```cpp

// Key types: std::expected, std::optional

```

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
