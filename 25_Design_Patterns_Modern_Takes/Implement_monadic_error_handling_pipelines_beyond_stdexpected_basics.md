# Implement monadic error handling pipelines beyond std::expected basics

**Category:** Design Patterns — Modern Takes  
**Item:** #751  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

Beyond basic `and_then`/`transform`, this file explores **advanced monadic patterns**: composing heterogeneous error types, building retry/fallback combinators, and creating domain-specific pipeline DSLs on top of `std::expected`.

### Monadic Operations Summary

| Method | Signature (on `expected<T,E>`) | When called | Returns |
| --- | --- | --- | --- |
| `and_then(f)` | `f: T → expected<U,E>` | On success | `expected<U,E>` |
| `transform(f)` | `f: T → U` | On success | `expected<U,E>` |
| `or_else(f)` | `f: E → expected<T,E2>` | On error | `expected<T,E2>` |
| `transform_error(f)` | `f: E → E2` | On error | `expected<T,E2>` |

### Key Insight — `or_else` for Fallbacks

```cpp

Pipeline:                     or_else fallback:
  step1()                       try primary
    .and_then(step2)              .or_else(try_secondary)
    .and_then(step3)              .or_else(try_tertiary)
    .transform(format)            .transform(use_result)

```

---

## Self-Assessment

### Q1: Chain a sequence of fallible operations using and_then without any explicit if-checks

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <expected>
#include <charconv>

// ═══════════ Rich error type ═══════════
struct PipelineError {
    std::string stage;
    std::string message;
};

// ═══════════ Five fallible steps — no if-checks anywhere ═══════════
auto read_input(const std::string& raw)
    -> std::expected<std::string, PipelineError>
{
    if (raw.empty())
        return std::unexpected(PipelineError{"read", "empty input"});
    return raw;
}

auto trim(std::string s)
    -> std::expected<std::string, PipelineError>
{
    auto start = s.find_first_not_of(" \t");
    if (start == std::string::npos)
        return std::unexpected(PipelineError{"trim", "whitespace only"});
    return s.substr(start, s.find_last_not_of(" \t") - start + 1);
}

auto parse_int(const std::string& s)
    -> std::expected<int, PipelineError>
{
    int value{};
    auto [ptr, ec] = std::from_chars(s.data(), s.data() + s.size(), value);
    if (ec != std::errc{})
        return std::unexpected(PipelineError{"parse", "not a valid integer"});
    return value;
}

auto validate_range(int v)
    -> std::expected<int, PipelineError>
{
    if (v < 1 || v > 100)
        return std::unexpected(PipelineError{"validate", "out of range [1,100]"});
    return v;
}

auto compute(int v)
    -> std::expected<double, PipelineError>
{
    return static_cast<double>(v * v) / 2.0;
}

int main() {
    // Five chained steps — zero if-checks!
    auto result = read_input("  42  ")
        .and_then(trim)
        .and_then(parse_int)
        .and_then(validate_range)
        .and_then(compute);

    if (result)
        std::cout << "Result: " << *result << '\n';  // 882.0
    else
        std::cout << "[" << result.error().stage << "] "
                  << result.error().message << '\n';

    // Test error path
    auto bad = read_input("  abc  ")
        .and_then(trim)
        .and_then(parse_int)      // Fails here
        .and_then(validate_range)  // Skipped
        .and_then(compute);        // Skipped

    std::cout << "[" << bad.error().stage << "] "
              << bad.error().message << '\n';
    // [parse] not a valid integer
}

```

### Q2: Use transform_error to enrich an error type as it propagates up the call stack

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <expected>
#include <vector>

// ═══════════ Layered error types ═══════════
struct LowLevelError {
    int code;
    std::string detail;
};

struct AppError {
    std::string layer;
    std::string message;
    std::vector<std::string> context;  // Accumulated context as error travels up
};

// Low-level function returns low-level error
auto fetch_data(int id)
    -> std::expected<std::string, LowLevelError>
{
    if (id < 0)
        return std::unexpected(LowLevelError{-1, "negative id"});
    if (id > 1000)
        return std::unexpected(LowLevelError{404, "not found"});
    return "data_for_" + std::to_string(id);
}

int main() {
    auto result = fetch_data(-5)
        // Convert low-level error → app error
        .transform_error([](LowLevelError e) -> AppError {
            return AppError{"storage", "fetch failed: " + e.detail, {}};
        })
        // Enrich with request context
        .transform_error([](AppError e) -> AppError {
            e.context.push_back("during user profile load");
            return e;
        })
        // Enrich with handler context
        .transform_error([](AppError e) -> AppError {
            e.context.push_back("in GET /api/users/-5");
            e.layer = "http";
            return e;
        });

    if (!result) {
        auto& err = result.error();
        std::cout << "[" << err.layer << "] " << err.message << '\n';
        for (auto& ctx : err.context)
            std::cout << "  context: " << ctx << '\n';
    }
    // Output:
    // [http] fetch failed: negative id
    //   context: during user profile load
    //   context: in GET /api/users/-5
}

```

### Q3: Implement a pipeline that logs errors at the top level without scattered error-check boilerplate

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <expected>
#include <functional>

struct Error { std::string msg; };
using Result = std::expected<int, Error>;

// ═══════════ or_else for fallback + logging ═══════════
auto primary_source() -> Result {
    return std::unexpected(Error{"primary DB down"});
}
auto fallback_cache() -> Result {
    return std::unexpected(Error{"cache miss"});
}
auto fallback_default() -> Result {
    return 42;  // Hard-coded default
}

int main() {
    // Pipeline with fallbacks using or_else
    auto result = primary_source()
        .or_else([](Error e) -> Result {
            std::cout << "  [warn] " << e.msg << ", trying cache...\n";
            return fallback_cache();
        })
        .or_else([](Error e) -> Result {
            std::cout << "  [warn] " << e.msg << ", using default...\n";
            return fallback_default();
        });

    // Single top-level error check
    if (result)
        std::cout << "Got value: " << *result << '\n';
    else
        std::cout << "All sources failed: " << result.error().msg << '\n';

    // Output:
    //   [warn] primary DB down, trying cache...
    //   [warn] cache miss, using default...
    // Got value: 42
}

```

---

## Notes

- `or_else` is the "error-side `and_then`" — use it for fallback chains and error recovery
- `transform_error` is pure mapping (can't change success/failure status); `or_else` can recover
- All four monadic methods short-circuit: success methods skip on error, error methods skip on success
- Compile with `-std=c++23`; GCC 13+, Clang 17+, MSVC 19.37+ support the monadic interface
