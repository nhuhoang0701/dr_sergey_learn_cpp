# Implement railway-oriented programming for error handling

**Category:** Functional Programming Patterns  
**Standard:** C++23  
**Reference:** <https://fsharpforfunandprofit.com/rop/>  

---

## Topic Overview

Railway-oriented programming (ROP) models computation as two parallel "tracks": success and failure. Functions either continue on the success track or switch to the error track. Once on the error track, subsequent operations are skipped automatically.

The mental model is exactly what the name suggests: imagine two parallel railway tracks. Your value starts on the success track. Each operation is a switch - it either keeps the value on the success track, or diverts it to the error track. Once diverted, the train stays on the error track and all subsequent switches are bypassed. You only collect the final result at the end of the line.

### The Railway Metaphor

```cpp
Success track:  --[parse]--[validate]--[save]--> result
                     |          |         |
Error track:    -----+----------+---------+-----> error
```

### Implementation with std::expected

`std::expected` maps directly onto the two-track model: the `T` value lives on the success track, and the `E` error lives on the error track. The C++23 monadic operations (`and_then`, `transform`) implement the track-switching logic:

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

Notice how clean the pipeline is to read: parse, then validate, then format. Each step knows nothing about the others. If any step fails, the error is automatically carried to the end - no explicit propagation code needed.

### Collecting Multiple Errors

The basic ROP pattern stops at the first error. Sometimes you want to collect all errors that occur, not just the first one. For form validation, for example, it's much friendlier to show all the problems at once rather than one at a time. Here's how to do that with a custom error accumulator:

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

This approach validates everything eagerly and accumulates all failures before returning. The result either contains the valid input or the full list of errors.

---

## Self-Assessment

### Q1: How does ROP differ from exception-based error handling

ROP makes error paths explicit in the type system (`Result<T>` vs plain `T`). Errors are values, not control flow events. You can compose error-handling pipelines, inspect errors at each step, and the compiler forces you to handle them. Exceptions are implicit - callers don't know a function can fail just from its signature. ROP also has no performance cost from stack unwinding; the error is just a value that gets forwarded.

### Q2: What is `or_else` in ROP terms

`or_else` is a "recovery switch" - when the value is on the error track, it tries to switch back to the success track. For example: `parse_int(s).or_else([](auto&) -> Result<int> { return 0; })` recovers with a default value. If the value is already on the success track, `or_else` is a no-op - the success value passes through unchanged.

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

Rust's `?` desugars to early return on error, which produces sequential-looking code. The C++23 version instead chains the operations as a pipeline. Both express the same logic - run each step, stop on error, forward the error to the caller - just with different syntactic shapes.

---

## Notes

- ROP is built from the `and_then`/`transform`/`or_else` trio on monadic types.
- C++23 `std::expected` with monadic operations is the standard building block for ROP in modern C++.
- For pre-C++23 code, use Boost.Outcome or tl::expected, which provide the same monadic operations.
- ROP shines for multi-step validation and transformation pipelines where each step can independently fail with a meaningful error.
