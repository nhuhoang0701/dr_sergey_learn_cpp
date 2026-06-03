# Know std::expected vs Boost.Outcome vs Boost.LEAF comparison

**Category:** Error Handling  
**Standard:** C++23  
**Reference:** <https://www.boost.org/doc/libs/release/libs/outcome/>  

---

## Topic Overview

Three major result/error types exist in the C++ ecosystem. Each has different design goals, and choosing the wrong one for your situation creates unnecessary friction. The short version: `std::expected` is the right starting point for most new C++23 code, Boost.Outcome is useful when you need a third state (exception alongside value and error), and Boost.LEAF is the specialist's tool for environments where error-path performance is the top priority.

### Comparison

| Feature | `std::expected<T,E>` | Boost.Outcome | Boost.LEAF |
| --- | --- | --- | --- |
| Standard | C++23 | Boost | Boost |
| Error type | One error type E | Value + Error + Exception | Error in TLS |
| Size | `max(T, E) + tag` | `max(T, E, exception_ptr)` | Just T (errors in TLS) |
| Monadic ops | Yes (C++23) | Yes | No (different model) |
| Performance | Good | Good | Excellent (zero-cost transport) |
| Exception interop | Manual | Built-in | Built-in |

### std::expected (Standard, Simple)

`std::expected` is the recommended choice for new C++23 code. You get a clean value-or-error type with monadic operations (`.transform()`, `.and_then()`, `.or_else()`) for composing operations without nested `if` checks. Here's the core pattern.

```cpp
#include <expected>
std::expected<int, std::string> parse(const std::string& s) {
    try { return std::stoi(s); }
    catch (...) { return std::unexpected("parse error"); }
}
auto result = parse("42").transform([](int x) { return x * 2; });
```

If `parse` returns an error, `.transform` passes the error through untouched. If it succeeds, the lambda runs on the value. No `if` checks needed at each step.

### Boost.Outcome (Rich, Three-State)

Boost.Outcome was a direct precursor to `std::expected` and has a richer model. Where `std::expected` holds either a value or an error, Outcome's `outcome<T,E,EP>` can hold a value, an error, or an exception pointer - all as distinct states. The `BOOST_OUTCOME_TRY` macro acts like Rust's `?` operator: it returns early from the function on error, eliminating boilerplate.

```cpp
#include <boost/outcome.hpp>
namespace outcome = BOOST_OUTCOME_V2_NAMESPACE;

// result<T, E> = value | error
// outcome<T, E, EP> = value | error | exception_ptr
outcome::result<int> parse(const std::string& s) {
    if (s.empty()) return outcome::failure(make_error_code(std::errc::invalid_argument));
    return std::stoi(s);
}

// BOOST_OUTCOME_TRY: macro for early return (like Rust's ?)
outcome::result<int> process(const std::string& s) {
    BOOST_OUTCOME_TRY(auto val, parse(s));  // Returns on error
    return val * 2;
}
```

The `BOOST_OUTCOME_TRY` line is concise: if `parse(s)` returns an error, `process` immediately returns that error. If it succeeds, `val` gets the value and the function continues. This eliminates a lot of `if (!result) return result.error();` boilerplate.

### Boost.LEAF (Zero-Cost Error Transport)

LEAF takes a fundamentally different approach. Instead of storing error details inside the return value, LEAF stores them in thread-local storage. The return type is just `leaf::result<T>`, which is roughly the size of `T` plus a bool. Error objects are only materialized at the handler site, and if no handler is registered for a particular error type, it's never constructed at all. This makes LEAF ideal for embedded systems or any context where error-path overhead needs to be genuinely zero.

```cpp
#include <boost/leaf.hpp>
namespace leaf = boost::leaf;

// Errors are stored in thread-local storage, not in the return value
leaf::result<int> parse(const std::string& s) {
    if (s.empty())
        return leaf::new_error(e_parse_error{"empty input"});
    return std::stoi(s);
}

// Handler at the call site:
int safe_parse(const std::string& s) {
    return leaf::try_handle_all(
        [&]() -> leaf::result<int> { return parse(s); },
        [](e_parse_error const& e) { return -1; },           // Handle parse error
        [](leaf::catch_<std::exception> const& e) { return -2; }, // Handle exception
        []() { return -3; }                                    // Catch-all
    );
}
```

The trade-off is that LEAF's model is less composable than `std::expected` - you can't just chain `.transform()` calls. You work with `try_handle_all` blocks instead. For most application code that's awkward; for high-performance library code on resource-constrained systems, it's the right tool.

---

## Self-Assessment

### Q1: When to choose each

- **std::expected**: New code, standard conformance, simple value-or-error. This is the default choice for C++23 and later.
- **Boost.Outcome**: Need three states (value/error/exception), or need a `TRY`-style early-return macro before C++ adopts one natively.
- **Boost.LEAF**: Maximum performance on the error path, errors carry large payloads, embedded systems, or `-fno-exceptions` environments.

### Q2: How does LEAF achieve zero-cost transport

LEAF stores error objects in thread-local storage, not in the return value. The return type is just `result<T>` (size of `T + bool`). Error details are loaded only at the handler site. If no handler needs the error info, it's never constructed. The reason this matters is that in some hot code paths you might want to propagate error status through many layers without ever paying to copy or construct the error payload - LEAF makes that possible.

### Q3: Can you mix these with exceptions

All three interoperate with exceptions. `expected` can store `exception_ptr` as the error type. Outcome has a three-state `outcome<T,E,EP>` that explicitly models the exception case. LEAF can catch both result errors and exceptions in the same handler block, which is convenient when you're wrapping code that mixes the two styles.

---

## Notes

- `std::expected` is the standardized, recommended choice for C++23+ code.
- Boost.Outcome was a precursor to `std::expected` but has a richer model when three states are needed.
- Boost.LEAF is optimal for embedded and `-fno-exceptions` environments where error payload cost must be zero.
- Consider `BOOST_OUTCOME_TRY` as a stopgap until C++ gets a native `?` operator for result propagation.
