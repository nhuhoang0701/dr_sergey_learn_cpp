# Implement railway-oriented programming for error handling

**Category:** Functional Programming Patterns  
**Standard:** C++23  
**Reference:** <https://fsharpforfunandprofit.com/rop/>  

---

## Topic Overview

Railway-oriented programming (ROP) models computation as two parallel "tracks": success and failure. Functions either continue on the success track or switch to the error track. Once on the error track, subsequent operations are skipped.

### The Railway Metaphor

```cpp

Success track:  ──[parse]──[validate]──[save]──▶ result
                     │          │         │
Error track:    ─────┴──────────┴─────────┴────▶ error

```

### Implementation with std::expected

```cpp

#include <expected>
#include <string>
#include <iostream>

using Error = std::string;

template<typename T>
using Result = std::expected<T, Error>;

// Railway switches: functions that can fail
Result<int> parse_age(const std::string& input) {
    try {
        int age = std::stoi(input);
        return age;
    } catch (...) {
        return std::unexpected("Invalid number: " + input);
    }
}

Result<int> validate_age(int age) {
    if (age < 0 || age > 150)
        return std::unexpected("Age out of range: " + std::to_string(age));
    return age;
}

Result<std::string> format_age(int age) {
    return "User is " + std::to_string(age) + " years old";
}

// Railway pipeline — C++23 monadic operations:
Result<std::string> process_age(const std::string& input) {
    return parse_age(input)           // May switch to error track
        .and_then(validate_age)       // Skipped if already on error track
        .and_then(format_age);        // Skipped if already on error track
}

int main() {
    auto r1 = process_age("25");
    auto r2 = process_age("abc");
    auto r3 = process_age("999");

    std::cout << r1.value_or("ERROR") << "\n";  // "User is 25 years old"
    std::cout << r2.value_or("ERROR") << "\n";  // "ERROR"
    std::cout << r3.error() << "\n";             // "Age out of range: 999"
}

```

### Collecting Multiple Errors

```cpp

#include <expected>
#include <string>
#include <vector>

struct ValidationErrors {
    std::vector<std::string> errors;
    void add(std::string e) { errors.push_back(std::move(e)); }
    bool empty() const { return errors.empty(); }
};

template<typename T>
using Validated = std::expected<T, ValidationErrors>;

struct UserInput { std::string name; std::string age; std::string email; };

Validated<UserInput> validate_all(const UserInput& input) {
    ValidationErrors errs;
    if (input.name.empty()) errs.add("Name is required");
    if (input.age.empty()) errs.add("Age is required");
    if (input.email.find('@') == std::string::npos) errs.add("Invalid email");
    if (!errs.empty()) return std::unexpected(std::move(errs));
    return input;
}

```

---

## Self-Assessment

### Q1: How does ROP differ from exception-based error handling

ROP makes error paths explicit in the type system (`Result<T>` vs `T`). Errors are values, not control flow. You can compose error-handling pipelines, inspect errors at each step, and the compiler forces you to handle errors. Exceptions are implicit — callers don't know a function can fail from its signature.

### Q2: What is `or_else` in ROP terms

`or_else` is a "recovery switch" — when on the error track, it tries to switch back to the success track. Example: `parse_int(s).or_else([](auto&) -> Result<int> { return 0; })` recovers with a default value. If already on the success track, `or_else` is a no-op.

### Q3: Compare with Rust's `?` operator

```rust

// Rust:
fn process(input: &str) -> Result<String, Error> {
    let age = parse_age(input)?;      // Returns early on error
    let valid = validate_age(age)?;
    Ok(format_age(valid))
}

// C++23 equivalent:
Result<std::string> process(const std::string& input) {
    return parse_age(input)
        .and_then(validate_age)
        .and_then(format_age);
}

```

---

## Notes

- ROP is the `and_then`/`transform`/`or_else` trio on monadic types.
- C++23 `std::expected` with monadic operations is the standard ROP building block.
- For pre-C++23, use Boost.Outcome or tl::expected which provide the same operations.
- ROP shines for multi-step validation and transformation pipelines.
