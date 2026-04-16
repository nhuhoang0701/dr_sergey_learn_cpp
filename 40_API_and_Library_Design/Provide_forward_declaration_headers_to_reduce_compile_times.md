# Provide forward declaration headers to reduce compile times

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#sf-source-files>  

---

## Topic Overview

Heavy headers slow compilation. Forward declaration headers (`_fwd.h`) let consumers declare types without including full definitions.

### Pattern

```cpp

// mylib/types_fwd.hpp — lightweight, include anywhere
#pragma once
namespace mylib {
    class Connection;
    class Request;
    class Response;
    struct Config;
    enum class Status : int;
}

// mylib/types.hpp — full definitions, include only in .cpp
#pragma once
#include "types_fwd.hpp"
#include <string>
#include <vector>
#include <memory>

namespace mylib {
    class Connection {
        // ... heavy implementation
    };
    // ...
}

// consumer.hpp — can use forward declarations only
#include <mylib/types_fwd.hpp>
#include <memory>

class Consumer {
    std::unique_ptr<mylib::Connection> conn_;  // OK: unique_ptr of incomplete type
public:
    void process(const mylib::Request& req);   // OK: reference to incomplete type
};

// consumer.cpp — needs full definitions for method bodies
#include "consumer.hpp"
#include <mylib/types.hpp>

void Consumer::process(const mylib::Request& req) {
    // Now Connection and Request are complete
}

```

### What Can Use Incomplete Types

| Usage | Incomplete OK? |
| --- | --- |
| `T*`, `T&`, `const T&` | Yes |
| `std::unique_ptr<T>` | Yes (delete in .cpp) |
| `std::shared_ptr<T>` | Yes |
| `std::vector<T>` | No (needs `sizeof(T)`) |
| `T` by value | No |
| `sizeof(T)` | No |
| Calling `T` methods | No |

---

## Self-Assessment

### Q1: Why does `unique_ptr<T>` work with incomplete types but `vector<T>` doesn't

`unique_ptr<T>` only stores a `T*` (pointer has fixed size). The deleter is called in the destructor, which can be in a .cpp file where `T` is complete. `vector<T>` needs `sizeof(T)` to allocate storage and needs `T`'s destructor to clean up — both require a complete type.

### Q2: Measure compile time improvement from forward declarations

```cpp

// Before: consumer.hpp includes full mylib/types.hpp
// Including 50 headers transitively → 200ms per TU

// After: consumer.hpp includes only mylib/types_fwd.hpp
// Including 2 headers → 20ms per TU

// With 100 TUs including consumer.hpp:
// Before: 100 × 200ms = 20s
// After:  100 × 20ms  = 2s (10x faster)

```

### Q3: When can't you use forward declarations

When the header needs the type's size (value members, inheritance), calls methods on it, or uses it as a template argument for containers that need complete types.

---

## Notes

- The standard library provides `<iosfwd>` as a forward declaration header for streams.
- Rule of thumb: headers should include as little as possible; .cpp files include what they need.
- Pimpl idiom + forward declarations = maximum compilation firewall.
