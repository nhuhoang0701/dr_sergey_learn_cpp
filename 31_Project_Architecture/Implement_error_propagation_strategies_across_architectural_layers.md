# Implement error propagation strategies across architectural layers

**Category:** Project Architecture

---

## Topic Overview

**Error propagation** is how errors travel from where they occur (infrastructure) to where they're handled (application/UI layer). Each layer should translate errors into its own vocabulary without leaking implementation details. C++ offers exceptions, error codes, `std::expected` (C++23), and result types for propagation.

### Error Propagation Strategies

| Strategy | Performance | Ergonomics | Cross-boundary | Use Case |
| --- | --- | --- | --- | --- |
| **Exceptions** | Overhead on throw | Natural (try/catch) | Problematic across DLLs | Business logic |
| **Error codes** | Zero overhead | Verbose, easy to ignore | Safe across ABI | C APIs, embedded |
| **std::expected** | Zero overhead | Clean (C++23) | Value type, safe | Modern layer APIs |
| **Result<T,E>** | Zero overhead | Monadic chaining | Value type, safe | Rust-inspired C++ |

---

## Self-Assessment

### Q1: Design layered error types with translation

**Answer:**

```cpp

#include <expected>
#include <string>
#include <variant>
#include <system_error>

// === Infrastructure layer errors ===
enum class DbError {
    ConnectionFailed,
    QueryFailed,
    Timeout,
    DuplicateKey,
    NotFound
};

enum class NetworkError {
    ConnectionRefused,
    Timeout,
    DnsResolution,
    TlsHandshake,
    ProtocolError
};

// === Domain layer errors (infrastructure-agnostic) ===
struct DomainError {
    enum class Kind {
        NotFound,
        AlreadyExists,
        ValidationFailed,
        Unauthorized,
        InternalError
    };
    Kind kind;
    std::string message;
    std::string field;  // For validation errors
};

// === Application layer errors (user-facing) ===
struct AppError {
    int code;             // HTTP-like status code
    std::string message;  // User-safe message
    std::string detail;   // Developer detail (not shown to user)
};

// === Infrastructure -> Domain translation ===
DomainError translate(DbError err, const std::string& context) {
    switch (err) {
        case DbError::NotFound:
            return {DomainError::Kind::NotFound,
                    context + " not found", ""};
        case DbError::DuplicateKey:
            return {DomainError::Kind::AlreadyExists,
                    context + " already exists", ""};
        case DbError::ConnectionFailed:
        case DbError::Timeout:
        case DbError::QueryFailed:
            return {DomainError::Kind::InternalError,
                    "Database unavailable", ""};
    }
    return {DomainError::Kind::InternalError, "Unknown error", ""};
}

// === Domain -> Application translation ===
AppError translate(const DomainError& err) {
    switch (err.kind) {
        case DomainError::Kind::NotFound:
            return {404, err.message, ""};
        case DomainError::Kind::AlreadyExists:
            return {409, err.message, ""};
        case DomainError::Kind::ValidationFailed:
            return {400, err.message, "field: " + err.field};
        case DomainError::Kind::Unauthorized:
            return {401, "Unauthorized", err.message};
        case DomainError::Kind::InternalError:
            return {500, "Internal server error", err.message};
    }
    return {500, "Unknown error", ""};
}

```

### Q2: Use std::expected for clean error propagation

**Answer:**

```cpp

// === Repository (infrastructure) ===
class UserRepository {
public:
    std::expected<User, DbError> find_by_id(int id) {
        auto result = db_.query("SELECT * FROM users WHERE id = ?", id);
        if (!result)
            return std::unexpected(DbError::QueryFailed);
        if (result->empty())
            return std::unexpected(DbError::NotFound);
        return parse_user(*result);
    }

    std::expected<void, DbError> save(const User& user) {
        if (!db_.execute("INSERT INTO users ...", user))
            return std::unexpected(DbError::QueryFailed);
        return {};
    }

private:
    Database db_;
};

// === Domain service (translates infrastructure errors) ===
class UserService {
public:
    explicit UserService(UserRepository& repo) : repo_(repo) {}

    std::expected<User, DomainError> get_user(int id) {
        auto result = repo_.find_by_id(id);
        if (!result)
            return std::unexpected(
                translate(result.error(), "User"));
        return *result;
    }

    std::expected<User, DomainError> create_user(
            const std::string& name, const std::string& email) {
        // Validation (domain rule)
        if (name.empty())
            return std::unexpected(DomainError{
                DomainError::Kind::ValidationFailed,
                "Name is required", "name"});

        User user{0, name, email, true};
        auto save_result = repo_.save(user);
        if (!save_result)
            return std::unexpected(
                translate(save_result.error(), "User"));
        return user;
    }

private:
    UserRepository& repo_;
};

// === Application handler (translates domain errors) ===
class UserHandler {
public:
    AppError handle_get_user(int id) {
        auto result = service_.get_user(id);
        if (!result)
            return translate(result.error());
        // Success: serialize user to response
        return {200, "OK", ""};
    }

private:
    UserService service_;
};

```

### Q3: Implement a Result type with monadic chaining (pre-C++23)

**Answer:**

```cpp

// === Result<T, E> for pre-C++23 ===
template<typename T, typename E>
class Result {
public:
    static Result ok(T value) { return Result(std::move(value)); }
    static Result err(E error) { return Result(std::move(error)); }

    bool is_ok() const { return std::holds_alternative<T>(data_); }
    bool is_err() const { return !is_ok(); }

    T& value() { return std::get<T>(data_); }
    const T& value() const { return std::get<T>(data_); }
    E& error() { return std::get<E>(data_); }
    const E& error() const { return std::get<E>(data_); }

    // Monadic: transform value if ok
    template<typename F>
    auto map(F&& fn) -> Result<decltype(fn(value())), E> {
        if (is_ok())
            return Result<decltype(fn(value())), E>::ok(fn(value()));
        return Result<decltype(fn(value())), E>::err(error());
    }

    // Monadic: chain operations that return Result
    template<typename F>
    auto and_then(F&& fn) -> decltype(fn(value())) {
        if (is_ok()) return fn(value());
        using RetType = decltype(fn(value()));
        return RetType::err(error());
    }

    // Map error type
    template<typename F>
    auto map_error(F&& fn) -> Result<T, decltype(fn(error()))> {
        if (is_err())
            return Result<T, decltype(fn(error()))>::err(fn(error()));
        return Result<T, decltype(fn(error()))>::ok(value());
    }

private:
    explicit Result(T val) : data_(std::move(val)) {}
    explicit Result(E err) : data_(std::move(err)) {}
    std::variant<T, E> data_;
};

// === Chained usage ===
auto process_order(int order_id) {
    return find_order(order_id)               // Result<Order, DbError>
        .map_error([](DbError e) {            // -> Result<Order, DomainError>
            return translate(e, "Order");
        })
        .and_then([](Order& o) {              // -> Result<Invoice, DomainError>
            return calculate_invoice(o);
        })
        .map([](Invoice& inv) {               // -> Result<string, DomainError>
            return serialize_invoice(inv);
        });
}

```

---

## Notes

- **Never leak infrastructure errors to the domain**: translate `DbError` -> `DomainError` at the repository boundary
- **Never leak domain details to users**: translate `DomainError` -> `AppError` at the handler boundary
- `std::expected<T,E>` (C++23) is the standard approach — use it when available
- For pre-C++23, use a custom `Result<T,E>` or `tl::expected` library
- **Exceptions vs expected**: exceptions for truly exceptional cases (out of memory); `expected` for business errors (not found, validation)
- Monadic chaining (`and_then`, `map`) avoids nested `if` checks
- Log the original error at the translation boundary for debugging; return a sanitized version upstream
