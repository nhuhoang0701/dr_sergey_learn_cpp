# Implement monadic error chaining with std::expected and std::optional

**Category:** Functional Programming Patterns  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

Monadic operations let you chain fallible computations without manual error checking at each step. C++23 adds `and_then`, `transform`, and `or_else` to both `std::optional` and `std::expected`.

### The Problem: Error Checking Boilerplate

```cpp

#include <expected>
#include <optional>
#include <string>
#include <charconv>
#include <system_error>

// Without monadic chaining — tedious, error-prone:
std::expected<int, std::string> parse_and_double_old(const std::string& input) {
    auto parsed = parse_int(input);
    if (!parsed) return std::unexpected(parsed.error());
    auto validated = validate_range(*parsed);
    if (!validated) return std::unexpected(validated.error());
    return *validated * 2;
}

```

### Monadic Chaining (C++23)

```cpp

#include <expected>
#include <optional>
#include <string>
#include <iostream>

std::expected<int, std::string> parse_int(const std::string& s) {
    try {
        return std::stoi(s);
    } catch (...) {
        return std::unexpected("parse error: " + s);
    }
}

std::expected<int, std::string> validate_range(int x) {
    if (x < 0 || x > 1000)
        return std::unexpected("out of range: " + std::to_string(x));
    return x;
}

// With monadic chaining — clean, composable:
auto process(const std::string& input) {
    return parse_int(input)
        .and_then(validate_range)          // Chain fallible operations
        .transform([](int x) { return x * 2; })  // Map over success value
        .or_else([](const std::string& err)       // Handle/recover from error
            -> std::expected<int, std::string> {
            std::cerr << "Warning: " << err << "\n";
            return 0;  // Default value on error
        });
}

int main() {
    auto r1 = process("42");    // 84
    auto r2 = process("abc");   // 0 (recovered)
    auto r3 = process("9999");  // 0 (recovered)
    std::cout << *r1 << " " << *r2 << " " << *r3 << "\n";
}

```

### std::optional Monadic Operations

```cpp

#include <optional>
#include <string>

std::optional<std::string> get_user(int id);
std::optional<std::string> get_address(const std::string& user);

// Chain optional operations:
auto get_user_city(int id) {
    return get_user(id)
        .and_then(get_address)                    // std::optional -> std::optional
        .transform([](const std::string& addr) {  // Transform the value
            return addr.substr(0, addr.find(','));
        })
        .value_or("Unknown");                     // Default if any step returned nullopt
}

```

---

## Self-Assessment

### Q1: What is the difference between `transform` and `and_then`

`transform(f)` applies `f` to the value and wraps the result: `optional<T>` → `optional<U>` (f returns U). `and_then(f)` applies `f` which itself returns an optional/expected: `optional<T>` → `optional<U>` (f returns `optional<U>`). `and_then` is flatMap/bind, `transform` is map/fmap.

### Q2: Implement `and_then` for a custom Result type

```cpp

template<typename T, typename E>
class Result {
    std::variant<T, E> data_;
public:
    template<typename F>
    auto and_then(F&& f) -> std::invoke_result_t<F, T> {
        if (auto* val = std::get_if<T>(&data_))
            return std::invoke(std::forward<F>(f), *val);
        return std::unexpected(std::get<E>(data_));
    }

    template<typename F>
    auto transform(F&& f) -> Result<std::invoke_result_t<F, T>, E> {
        if (auto* val = std::get_if<T>(&data_))
            return {std::invoke(std::forward<F>(f), *val)};
        return {std::get<E>(data_)};
    }
};

```

### Q3: How does `or_else` differ from a catch block

`or_else` operates on the error value and returns a new expected/optional — allowing recovery by providing a default value or converting the error. It's composable (chainable), whereas catch blocks are control flow statements. `or_else` can also transform the error type.

---

## Notes

- C++23 monadic operations work on both `std::optional` and `std::expected`.
- This is the C++ equivalent of Rust's `?` operator and `.map()/.and_then()`.
- Monadic chaining eliminates nested `if (!result)` boilerplate.
- `or_else` is for error recovery; `transform_error` is for error type conversion.
