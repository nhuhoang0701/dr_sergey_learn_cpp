# Use designated initializers (C++20) for self-documenting struct initialization

**Category:** Best Practices & Idioms  
**Item:** #140  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/aggregate_initialization>  

---

## Topic Overview

**Designated initializers** (C++20) let you name fields when initializing aggregates, making the code self-documenting and catching reordering bugs at compile time.

```cpp

// Before (C++17): what does true, false, 42 mean?
Config c{true, false, 42, 3.14f};

// After (C++20): crystal clear!
Config c{.verbose = true, .debug = false, .max_retries = 42, .timeout = 3.14f};

```

---

## Self-Assessment

### Q1: Rewrite positional initialization using designated initializers

```cpp

#include <iostream>
#include <string>

struct Config {
    bool verbose = false;
    bool debug = false;
    int max_retries = 3;
    float timeout = 5.0f;
    std::string host = "localhost";
};

struct NetworkRequest {
    std::string url;
    std::string method = "GET";
    int timeout_ms = 5000;
    bool follow_redirects = true;
    int max_retries = 3;
};

int main() {
    // BAD: positional — what does true, false, 42 mean?
    Config bad{true, false, 42, 3.14f, "server.com"};

    // GOOD: designated initializers — self-documenting!
    Config good{
        .verbose = true,
        .debug = false,
        .max_retries = 42,
        .timeout = 3.14f,
        .host = "server.com"
    };

    // Can skip fields (they use defaults):
    Config minimal{.verbose = true, .max_retries = 10};
    // .debug = false, .timeout = 5.0f, .host = "localhost" (defaults)

    // Great for complex structs:
    NetworkRequest req{
        .url = "https://api.example.com/data",
        .method = "POST",
        .timeout_ms = 3000,
        .max_retries = 5
    };

    std::cout << "Host: " << good.host << '\n';
    std::cout << "Retries: " << minimal.max_retries << '\n';
    std::cout << "URL: " << req.url << '\n';
}
// Expected output:
// Host: server.com
// Retries: 10
// URL: https://api.example.com/data

```

### Q2: Show that designated initializers catch accidental field reordering

```cpp

#include <iostream>

struct Point {
    int x;
    int y;
    int z;
};

int main() {
    // Positional: reordering x,y is silent bug
    Point p1{10, 20, 30};     // x=10, y=20, z=30
    Point p2{20, 10, 30};     // x=20, y=10 — swapped! No warning.

    // Designated: reordering is a COMPILE ERROR
    Point p3{.x = 10, .y = 20, .z = 30};  // OK

    // This would NOT compile (out of order):
    // Point p4{.y = 20, .x = 10, .z = 30};  // ERROR!
    // C++ requires designators in declaration order.

    // Also catches typos:
    // Point p5{.x = 10, .w = 20, .z = 30};  // ERROR: no member 'w'

    std::cout << "p3 = (" << p3.x << ", " << p3.y << ", " << p3.z << ")\n";
}
// Expected output:
// p3 = (10, 20, 30)

```

### Q3: Explain restrictions on designated initializers

Designated initializers work only with **aggregate types**:

```cpp

#include <iostream>

// AGGREGATE: no user-declared constructors, no private members,
// no virtual functions, no base classes (relaxed in C++17)
struct Aggregate {
    int x;
    int y;
    int z = 0;
};

// NOT an aggregate: has constructor
struct NonAggregate {
    int x, y;
    NonAggregate(int a, int b) : x(a), y(b) {}
};

int main() {
    // OK: Aggregate type
    Aggregate a{.x = 1, .y = 2};

    // ERROR: NonAggregate can't use designated initializers
    // NonAggregate na{.x = 1, .y = 2};  // won't compile

    std::cout << a.x << ' ' << a.y << ' ' << a.z << '\n';
}
// Expected output:
// 1 2 0

```

**Restrictions (C++ vs C):**

| Rule | C++ | C |
| --- | --- | --- |
| Must be in declaration order | Yes | No |
| Can skip members | Yes (use defaults) | Yes |
| Nested designators (`.a.b = 1`) | No | Yes |
| Array designators (`[0] = 1`) | No | Yes |
| Mix designated + positional | No | Yes |
| Requires aggregate type | Yes | Yes |

---

## Notes

- Designated initializers are a subset of C's designated initializers (C++ is stricter).
- Combine with default member initializers for a "named parameter" effect.
- Great for configuration structs, option bags, and builder-like patterns.
- Zero runtime overhead — the compiler generates identical code to positional initialization.
