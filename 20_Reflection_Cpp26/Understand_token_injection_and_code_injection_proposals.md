# Understand token injection and code injection proposals

**Category:** Reflection (C++26)  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996> <https://wg21.link/P3294>  

---

## Topic Overview

### Beyond Introspection: Code Generation

C++26 static reflection (P2996) enables **introspection** — querying properties of types, members, and enumerators at compile time. But the complementary feature is **code injection** — the ability to generate new code based on reflection results.

### Token Injection (P3294)

The **token injection** proposal allows injecting tokens (new declarations, statements, expressions) into the program based on compile-time computations:

```cpp

#include <meta>

// Conceptual syntax (subject to evolution):
struct Point {
    int x, y, z;
};

// Generate operator== automatically using reflection + injection
consteval void generate_equality(std::meta::info type) {
    auto members = std::meta::nonstatic_data_members_of(type);
    // Inject: friend bool operator==(const T& a, const T& b) {
    //     return a.x == b.x && a.y == b.y && a.z == b.z;
    // }
    queue_injection(^^{
        friend bool operator==(const [:\(type):] & a, const [:\(type):] & b) {
            return [:\(fold_and(members, [](auto m) {
                return ^^{ a.[:$(m):] == b.[:$(m):] };
            })):];
        }
    });
}

```

### The Splice Operator `[: ... :]`

The splice operator takes a reflection value and injects it back into code:

```cpp

constexpr auto refl = ^^int;   // reflect on 'int'
[:refl:] x = 42;               // splice: declares 'int x = 42'

constexpr auto member = /* reflection of a data member */;
obj.[:member:] = 10;           // splice: accesses that member on obj

```

### Token Sequences

Token sequences (`^^{ ... }`) represent fragments of code that can be manipulated at compile time:

```cpp

consteval auto make_getter(std::meta::info member) {
    auto name = std::meta::identifier_of(member);
    auto type = std::meta::type_of(member);

    return ^^{
        constexpr auto get_[:\(name):]() const -> const [:\(type):]& {
            return this->[:\(member):];
        }
    };
}

```

### Practical Example: Auto-Generating Struct Serialization

```cpp

#include <meta>
#include <string>

template<typename T>
consteval void inject_to_json() {
    auto members = std::meta::nonstatic_data_members_of(^^T);

    queue_injection(^^{
        friend std::string to_json(const T& obj) {
            std::string result = "{";
            bool first = true;
            template for (constexpr auto m : members) {
                if (!first) result += ", ";
                result += "\"";
                result += std::meta::identifier_of(m);
                result += "\": ";
                result += std::to_string(obj.[:m:]);
                first = false;
            }
            result += "}";
            return result;
        }
    });
}

struct Config {
    int width = 1920;
    int height = 1080;
    int fps = 60;
    consteval { inject_to_json<Config>(); }
};

// Now Config has: friend std::string to_json(const Config& obj);

```

### Code Injection vs. Macros vs. Templates

| Approach | Introspection | Type-safe | Debugging | Power |
| --- | :---: | :---: | :---: | :---: |
| Macros | No | No | Poor | High (textual) |
| Templates/SFINAE | Limited | Yes | OK | Medium |
| Concepts | Type constraints | Yes | Good | Medium |
| Reflection + Injection | Full | Yes | TBD | Very High |

---

## Self-Assessment

### Q1: What is the difference between reflection and injection

- **Reflection** (`^^` operator): Queries compile-time properties of existing code — member names, types, attributes.
- **Injection** (`queue_injection`, token sequences): Generates **new** code based on reflection results.

Reflection reads; injection writes. Together, they enable fully automated code generation.

### Q2: Show how to enumerate struct members and generate getters

```cpp

struct Player {
    std::string name;
    int health;
    float speed;

    // Inject getters for all members
    consteval {
        for (auto m : std::meta::nonstatic_data_members_of(^^Player)) {
            queue_injection(make_getter(m));
        }
    }
};

// Generated:
// const std::string& get_name() const;
// const int& get_health() const;
// const float& get_speed() const;

```

### Q3: Why are token sequences better than string-based code generation

Token sequences (`^^{ ... }`) are:

1. **Parsed by the compiler** — syntax errors are caught immediately.
2. **Type-checked** — spliced values must match their contexts.
3. **Hygienic** — names are resolved in the correct scope (no accidental shadowing).
4. **Debuggable** — the compiler can map generated code back to the injection site.

String-based generation (like macros) has none of these properties.

---

## Notes

- Token injection proposals are still evolving — syntax may change before C++26 is finalized.
- The `queue_injection()` function adds code to be injected at the end of the current evaluation context.
- Injection works in `consteval` blocks inside class definitions.
- This is the most powerful compile-time code generation feature C++ has ever had.
