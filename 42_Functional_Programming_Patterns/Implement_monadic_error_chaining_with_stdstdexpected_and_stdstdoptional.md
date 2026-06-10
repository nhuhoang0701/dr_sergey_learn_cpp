# Implement monadic error chaining with std::expected and std::optional

**Category:** Functional Programming Patterns  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

Monadic operations let you chain fallible computations without manually checking for errors at each step. C++23 adds `and_then`, `transform`, and `or_else` to both `std::optional` and `std::expected`. If you've ever written code that's mostly `if (!result) return error;` boilerplate, this is the feature that cleans that up.

The word "monadic" can sound intimidating, but the concept is simple: instead of unwrapping a result, checking if it's valid, and then passing the value along manually, you let the type do the plumbing. You describe what to do with the value if it's present - and the type takes care of the "do nothing if already failed" logic automatically.

### The Problem: Error Checking Boilerplate

Here's what multi-step fallible code looks like without monadic chaining. Every step must be checked, and any error must be manually propagated:

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

This pattern is repetitive and error-prone. It's easy to accidentally return the wrong error, dereference a failed result, or simply forget a check. Monadic chaining eliminates all of that.

### Monadic Chaining (C++23)

With C++23, you can express the same pipeline as a chain of method calls. Each step only runs if the previous one succeeded - otherwise the error is automatically forwarded to the end:

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

The reason this trips people up at first is the difference between `and_then` and `transform`. Think of it this way: if the next step can itself fail, use `and_then`; if it always succeeds, use `transform`. The rule is that `and_then` expects a function returning `expected<U, E>`, while `transform` expects a function returning plain `U`.

### std::optional Monadic Operations

The same monadic operations work with `std::optional` for cases where there's no error message, just the absence of a value. This is perfect for nullable lookups chained together:

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

If `get_user` returns `nullopt`, `and_then` short-circuits and the whole chain evaluates to `nullopt`. You never touch `get_address`. The `value_or` at the end gives you the final default.

---

## Self-Assessment

### Q1: What is the difference between `transform` and `and_then`

`transform(f)` applies `f` to the value and wraps the result: `optional<T>` -> `optional<U>` where `f` returns plain `U`. `and_then(f)` applies `f` which itself returns an optional/expected: `optional<T>` -> `optional<U>` where `f` returns `optional<U>`. In functional programming terms, `and_then` is flatMap/bind (for functions that can fail), while `transform` is map/fmap (for functions that always succeed).

### Q2: Implement `and_then` for a custom Result type

Here's how you'd implement these operations yourself on a simple `Result` class built on `std::variant`. This shows you exactly what C++23 is doing under the hood:

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

The pattern is the same in both methods: check if the value is present, call the function if so, otherwise forward the error. The difference is just in how the return type is formed.

### Q3: How does `or_else` differ from a catch block

`or_else` operates on the error value and returns a new expected/optional, allowing recovery by providing a default value or converting the error type. It's composable - you can chain it with other monadic operations. A `catch` block is a control flow statement, not a value - it interrupts the flow and cannot be composed into a pipeline. `or_else` can also transform the error type itself, something `catch` cannot do without re-throwing with a new type.

---

## Notes

- C++23 monadic operations work on both `std::optional` and `std::expected`.
- This is the C++ equivalent of Rust's `?` operator and `.map()/.and_then()` methods.
- Monadic chaining eliminates nested `if (!result)` boilerplate and makes the happy path easy to read.
- `or_else` is for error recovery; `transform_error` is for changing the error type while staying on the error track.
