# Design aggregate types and understand aggregate initialization rules

**Category:** OOP Design

---

## Topic Overview

An **aggregate** is a class/struct with no user-declared constructors, no private/protected non-static data members, no virtual functions, and no virtual base classes.

### Aggregate rules by standard version

| Feature | C++11/14 | C++17 | C++20 |
| --- | :---: | :---: | :---: |
| User-declared constructors | Not allowed | Not allowed | Not allowed |
| Inherited constructors | Disqualify | OK | OK |
| Brace/designated init | `{a, b, c}` | `{a, b, c}` | `{.x=1, .y=2}` |
| Base classes | None allowed | Allowed | Allowed |
| Default member init | No | **Yes** | **Yes** |

### Designated Initializers (C++20)

```cpp

struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool ssl = false;
};
Config c{.port = 443, .ssl = true};  // host uses default

```

---

## Self-Assessment

### Q1: Show aggregate initialization and its gotchas

**Answer:**

```cpp

#include <string>
#include <array>
#include <iostream>

struct Point { int x, y, z; };

struct ServerConfig {
    std::string host = "localhost";
    int port = 8080;
    int max_connections = 100;
    bool debug = false;
};

int main() {
    // Basic aggregate init
    Point p1{1, 2, 3};
    Point p2{1, 2};     // z = 0 (value-initialized)
    Point p3{};          // All zero

    // C++20 designated initializers
    ServerConfig cfg1{.port = 443, .debug = true};
    // host = "localhost" (default), max_connections = 100 (default)

    // GOTCHA: order must match declaration order
    // ServerConfig bad{.debug = true, .port = 443};  // ERROR!

    // GOTCHA: narrowing conversions are errors in braces
    // Point p4{1.5, 2.5, 3.5};  // ERROR: double → int narrows

    // Nested aggregates
    struct Line { Point start, end; };
    Line l1{{0,0,0}, {1,1,1}};      // Nested brace init
    Line l2{.start{1,2,3}, .end{4,5,6}};  // C++20

    // std::array is an aggregate!
    std::array<int, 4> arr{1, 2, 3, 4};

    std::cout << cfg1.host << ":" << cfg1.port << "\n";
    return 0;
}

```

### Q2: Show what disqualifies a type from being an aggregate

**Answer:**

```cpp

#include <iostream>

// ✔ Aggregate
struct Good {
    int x;
    int y = 10;  // Default member init OK since C++14
};

// ✘ NOT aggregate: user-declared constructor
struct Bad1 {
    int x;
    Bad1() = default;  // C++17/20: even "= default" disqualifies!
    // In C++11/14: "= default" was OK for aggregates
};

// ✘ NOT aggregate: private member
struct Bad2 {
private:
    int x;
public:
    int y;
};

// ✘ NOT aggregate: virtual function
struct Bad3 {
    virtual void foo() {}
    int x;
};

// ✔ Aggregate with base class (C++17+)
struct Base { int a; };
struct Derived : Base { int b; };
// Derived d{{42}, 99};  // C++17: base subobject in braces

// Check at compile time
#include <type_traits>
static_assert(std::is_aggregate_v<Good>);
// static_assert(std::is_aggregate_v<Bad1>);  // FAILS
static_assert(std::is_aggregate_v<Derived>);  // C++17+

int main() {
    Good g1{1, 2};     // Aggregate init
    Good g2{.x = 1};   // C++20 designated
    Derived d{{10}, 20};
    std::cout << d.a << " " << d.b << "\n";
    return 0;
}

```

### Q3: Use aggregates for configuration and structured bindings

**Answer:**

```cpp

#include <string>
#include <vector>
#include <iostream>
#include <optional>

// Aggregate as a named parameter object
struct ConnectionOpts {
    std::string host = "localhost";
    int port = 5432;
    std::string database = "mydb";
    std::string user = "postgres";
    std::optional<std::string> password;
    int timeout_ms = 5000;
    int max_retries = 3;
    bool ssl = false;
};

void connect(const ConnectionOpts& opts) {
    std::cout << "Connecting to " << opts.user << "@"
              << opts.host << ":" << opts.port
              << "/" << opts.database
              << " ssl=" << opts.ssl << "\n";
}

// Aggregate return + structured bindings
struct ParseResult {
    bool success;
    std::string value;
    int error_code;
};

ParseResult parse(const std::string& input) {
    if (input.empty())
        return {false, "", -1};
    return {true, input, 0};
}

int main() {
    // Designated initializers = readable configuration
    connect({
        .host = "db.production.com",
        .port = 5433,
        .database = "analytics",
        .user = "app_service",
        .password = "secret123",
        .ssl = true
    });

    // Structured bindings with aggregate return
    auto [ok, value, code] = parse("hello");
    if (ok) std::cout << "Parsed: " << value << "\n";
    return 0;
}

```

---

## Notes

- C++20 designated initializers are the best alternative to named parameters in C++
- Aggregates are initialized **in declaration order** — designated init must follow this order too
- `std::is_aggregate_v<T>` lets you check at compile time
- Brace initialization catches narrowing (`double` → `int`) — this is a feature, not a bug
- In C++20, `= default` on a constructor **disqualifies** a type from being an aggregate (breaking change from C++17)
- Aggregates + structured bindings = great for multi-return values
