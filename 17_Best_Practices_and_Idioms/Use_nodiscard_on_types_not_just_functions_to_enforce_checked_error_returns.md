# Use [[nodiscard]] on types (not just functions) to enforce checked error returns

**Category:** Best Practices & Idioms  
**Item:** #143  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/attributes/nodiscard>  

---

## Topic Overview

`[[nodiscard]]` on a **type** means *every function that returns that type* gets an automatic warning if the return value is discarded. This is more powerful than marking individual functions.

```cpp

[[nodiscard]] struct Error { int code; };

Error do_work();    // automatically [[nodiscard]]!
Error validate();   // automatically [[nodiscard]]!
// Every function returning Error must have its result checked.

```

---

## Self-Assessment

### Q1: Mark a `Result<T,E>` type as `[[nodiscard]]` and show the warning

```cpp

#include <iostream>
#include <string>
#include <variant>

template<typename T, typename E>
[[nodiscard("Error result must be checked")]]
struct Result {
    std::variant<T, E> value;

    bool is_ok() const { return std::holds_alternative<T>(value); }
    T& unwrap() { return std::get<T>(value); }
    E& error() { return std::get<E>(value); }

    static Result ok(T val) { return Result{std::move(val)}; }
    static Result err(E e) { return Result{std::move(e)}; }
};

struct IoError {
    int code;
    std::string message;
};

Result<int, IoError> parse_number(const std::string& s) {
    try {
        return Result<int, IoError>::ok(std::stoi(s));
    } catch (...) {
        return Result<int, IoError>::err({-1, "parse failed: " + s});
    }
}

int main() {
    // GOOD: checking the result
    auto r1 = parse_number("42");
    if (r1.is_ok())
        std::cout << "Parsed: " << r1.unwrap() << '\n';

    auto r2 = parse_number("abc");
    if (!r2.is_ok())
        std::cout << "Error: " << r2.error().message << '\n';

    // BAD: this line would trigger a warning:
    // parse_number("99");  // warning: ignoring return value of type 'Result'
    //                      // with attribute 'nodiscard': Error result must be checked
}
// Expected output:
// Parsed: 42
// Error: parse failed: abc

```

### Q2: Explain why `[[nodiscard("reason")]]` is better than plain `[[nodiscard]]`

```cpp

#include <iostream>

// Without message:
[[nodiscard]]
struct ErrorCode {
    int code;
};
// Warning: "ignoring return value of type 'ErrorCode'"
// Not very helpful. WHY must I check it?

// With message (C++20):
[[nodiscard("Ignoring error codes leads to silent failures")]]
struct StatusCode {
    int code;
    bool is_ok() const { return code == 0; }
};
// Warning: "ignoring return value of type 'StatusCode':
//           Ignoring error codes leads to silent failures"
// Much clearer!

StatusCode connect(const char* host) {
    return StatusCode{0};
}

[[nodiscard("Memory leak if allocation result is discarded")]]
struct Buffer {
    int* data;
    size_t size;
};

Buffer allocate(size_t n) {
    return Buffer{new int[n], n};
}

int main() {
    auto s = connect("localhost");  // OK: result used
    if (s.is_ok())
        std::cout << "Connected\n";

    auto b = allocate(100);  // OK: result used
    std::cout << "Allocated " << b.size << " ints\n";
    delete[] b.data;

    // connect("server");  // WARNING with custom message!
    // allocate(50);       // WARNING: Memory leak if discarded!
}
// Expected output:
// Connected
// Allocated 100 ints

```

### Q3: Show how to suppress a `[[nodiscard]]` warning with `(void)` cast

```cpp

#include <iostream>

[[nodiscard("Check the error code")]]
struct Status {
    int code;
};

Status initialize() { return {0}; }
Status cleanup() { return {0}; }

int main() {
    // Method 1: Use the result (preferred)
    auto s = initialize();
    if (s.code != 0) return 1;

    // Method 2: Intentional discard with (void) cast
    // "I know this can fail, but I don't care in this context"
    (void)cleanup();  // suppresses the warning

    // Method 3: [[maybe_unused]] on the variable
    [[maybe_unused]] auto s2 = cleanup();

    // Method 4: std::ignore (C++17, for structured bindings)
    // auto [code] = cleanup();

    std::cout << "Done\n";
}
// Expected output:
// Done

```

---

## Notes

- `[[nodiscard]]` on types is available since C++17; the `("reason")` form since C++20.
- Apply to: error types, RAII guards, allocation results, lock types.
- The standard library uses it: `std::expected`, `std::error_code`, `std::unique_lock`.
- `[[nodiscard]]` on constructors (C++20) warns if a temporary is created and discarded.
