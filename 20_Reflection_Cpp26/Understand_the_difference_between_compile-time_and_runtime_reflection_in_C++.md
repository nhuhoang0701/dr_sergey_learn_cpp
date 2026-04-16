# Understand the difference between compile-time and runtime reflection in C++

**Category:** Reflection (C++26)  
**Item:** #538  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/meta>  

---

## Topic Overview

C++ has two reflection mechanisms: **RTTI** (runtime, since C++98) and **static reflection** (compile-time, C++26). They serve different purposes.

| Feature | RTTI (Runtime) | Static Reflection (Compile-time) |
| --- | --- | --- |
| When | Runtime | Compile time only |
| Overhead | vtable pointer + type_info | Zero runtime cost |
| Member enumeration | No | Yes (`members_of`) |
| Type names | `typeid(x).name()` | `identifier_of(^T)` |
| Downcasting | `dynamic_cast<D*>(b)` | N/A (no runtime types) |
| Requires | `-frtti`, polymorphic types | C++26 compiler |
| Data stored | Runtime type tables in binary | Nothing in binary |

---

## Self-Assessment

### Q1: Why C++26 reflection is purely compile-time with no runtime tables

```cpp

// C++26 with P2996 reflection
#include <meta>
#include <iostream>

struct Point { int x, y; };

int main() {
    // All reflection happens during compilation:
    constexpr auto info = ^Point;
    constexpr auto name = std::meta::identifier_of(info);
    constexpr auto count = std::meta::nonstatic_data_members_of(info).size();

    // These are compile-time constants:
    static_assert(name == "Point");
    static_assert(count == 2);

    // The compiled binary contains NO type tables for Point.
    // Compare with RTTI:
    // - typeid(Point) stores a type_info object in the binary
    // - dynamic_cast reads vtable pointers at runtime
    // - static reflection: ZERO runtime data

    // Proof: sizeof doesn't change, no vtable needed:
    static_assert(sizeof(Point) == 2 * sizeof(int));  // just data, no hidden ptr

    // You can verify: compile with -fno-rtti and reflection still works.
    // RTTI requires -frtti and polymorphic types (virtual functions).

    std::cout << name << " has " << count << " fields\n";
    // Output: Point has 2 fields
}

```

### Q2: Compare C++26 reflection with RTTI

```cpp

#include <meta>
#include <iostream>
#include <typeinfo>
#include <string_view>

// ===== RTTI approach (runtime) =====
struct Base { virtual ~Base() = default; };
struct Dog : Base { std::string name; };
struct Cat : Base { int lives; };

void identify_rtti(Base* b) {
    // Runtime cost: reads vtable, compares type_info
    if (auto* d = dynamic_cast<Dog*>(b)) {
        std::cout << "Dog: " << d->name << '\n';
    } else if (auto* c = dynamic_cast<Cat*>(b)) {
        std::cout << "Cat with " << c->lives << " lives\n";
    }
    // typeid: runtime name (mangled, implementation-defined):
    std::cout << "RTTI name: " << typeid(*b).name() << '\n';
}

// ===== Static reflection approach (compile-time) =====
struct Sensor { std::string id; double value; int unit; };

template <typename T>
void describe() {
    // ALL of this resolves at compile time:
    constexpr auto members = std::meta::nonstatic_data_members_of(^T);
    std::cout << std::meta::identifier_of(^T) << ":\n";
    template for (constexpr auto m : members) {
        std::cout << "  " << std::meta::identifier_of(m)
                  << ": " << std::meta::display_string_of(std::meta::type_of(m))
                  << '\n';
    }
}

int main() {
    // RTTI: works at runtime with polymorphic types
    Dog d; d.name = "Rex";
    identify_rtti(&d);

    // Static reflection: works at compile time with any type
    describe<Sensor>();
    // Output:
    // Sensor:
    //   id: std::string
    //   value: double
    //   unit: int

    // Key differences:
    // RTTI: requires virtual, adds vtable overhead, runtime cost
    // Reflection: no virtual needed, zero overhead, compile-time only
}

```

### Q3: Bootstrap runtime dispatch from compile-time reflection data

```cpp

#include <meta>
#include <iostream>
#include <string_view>
#include <functional>
#include <unordered_map>

enum class MsgType { Login, Logout, Ping, Data };

// Build a runtime dispatch table from compile-time reflection:
auto make_handler_map() {
    std::unordered_map<std::string_view, std::function<void()>> handlers;

    // Iterate enumerators at compile time, populate map at runtime:
    template for (constexpr auto e : std::meta::enumerators_of(^MsgType)) {
        constexpr auto name = std::meta::identifier_of(e);
        handlers[name] = [&]{
            std::cout << "Handling: " << name << '\n';
        };
    }
    return handlers;
}

// Another example: reflection-generated visitor
struct Config {
    std::string host;
    int port;
    bool debug;
};

void apply_from_map(Config& cfg,
                    const std::unordered_map<std::string, std::string>& kvmap) {
    template for (constexpr auto m : std::meta::nonstatic_data_members_of(^Config)) {
        constexpr auto name = std::meta::identifier_of(m);
        if (auto it = kvmap.find(std::string(name)); it != kvmap.end()) {
            using MType = [:std::meta::type_of(m):];
            if constexpr (std::is_same_v<MType, std::string>) {
                cfg.[:m:] = it->second;
            } else if constexpr (std::is_same_v<MType, int>) {
                cfg.[:m:] = std::stoi(it->second);
            } else if constexpr (std::is_same_v<MType, bool>) {
                cfg.[:m:] = (it->second == "true");
            }
        }
    }
}

int main() {
    // Runtime dispatch from compile-time enum reflection:
    auto handlers = make_handler_map();
    handlers["Login"]();   // Handling: Login
    handlers["Ping"]();    // Handling: Ping

    // Runtime config from compile-time struct reflection:
    Config cfg;
    apply_from_map(cfg, {{"host", "localhost"}, {"port", "8080"}, {"debug", "true"}});
    std::cout << cfg.host << ":" << cfg.port
              << " debug=" << cfg.debug << '\n';
    // Output: localhost:8080 debug=1
}

```

---

## Notes

- C++26 reflection **replaces** most RTTI use cases with zero-overhead alternatives.
- RTTI is still needed for runtime polymorphic downcasting (`dynamic_cast`).
- Static reflection works with `-fno-rtti` — no virtual functions required.
- Use `template for` to bridge compile-time reflection to runtime data structures.
- Compile-time reflection data can seed runtime dispatch tables, serializers, validators, etc.
