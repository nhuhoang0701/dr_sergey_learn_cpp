# Know std::expected vs Boost.Outcome vs Boost.LEAF comparison

**Category:** Error Handling  
**Standard:** C++23  
**Reference:** <https://www.boost.org/doc/libs/release/libs/outcome/>  

---

## Topic Overview

Three major result/error types exist in the C++ ecosystem. Each has different design goals.

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

```cpp

#include <expected>
std::expected<int, std::string> parse(const std::string& s) {
    try { return std::stoi(s); }
    catch (...) { return std::unexpected("parse error"); }
}
auto result = parse("42").transform([](int x) { return x * 2; });

```

### Boost.Outcome (Rich, Three-State)

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

### Boost.LEAF (Zero-Cost Error Transport)

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

---

## Self-Assessment

### Q1: When to choose each

- **std::expected**: New code, standard conformance, simple value-or-error
- **Boost.Outcome**: Need three states (value/error/exception), need `TRY` macro
- **Boost.LEAF**: Maximum performance, errors are large, embedded systems

### Q2: How does LEAF achieve zero-cost transport

LEAF stores error objects in thread-local storage, not in the return value. The return type is just `result<T>` (size of `T + bool`). Error details are loaded only at the handler site. If no handler needs the error info, it's never constructed.

### Q3: Can you mix these with exceptions

All three interoperate with exceptions. `expected` can store `exception_ptr` as the error type. Outcome has a three-state `outcome<T,E,EP>`. LEAF can catch both result errors and exceptions in the same handler.

---

## Notes

- `std::expected` is the standardized, recommended choice for C++23+ code.
- Boost.Outcome was a precursor to `std::expected` but has a richer model.
- Boost.LEAF is optimal for embedded and `-fno-exceptions` environments.
- Consider `BOOST_OUTCOME_TRY` as a stopgap until C++ gets a native `?` operator.
