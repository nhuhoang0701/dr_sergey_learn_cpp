# Use static_assert with concept-based diagnostics for better error messages

**Category:** Compile-Time Programming  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/static_assert>  

---

## Topic Overview

`static_assert` has been available since C++11, but C++20 concepts dramatically improve the quality of error messages by testing each requirement individually.

### Basic static_assert

```cpp

#include <type_traits>

template<typename T>
class Container {
    static_assert(!std::is_reference_v<T>,
                  "Container does not support reference types. Use std::reference_wrapper.");
    static_assert(std::is_destructible_v<T>,
                  "T must be destructible.");
    static_assert(!std::is_void_v<T>,
                  "Container<void> is not supported.");
};

// Container<int&> →
// error: static assertion failed: Container does not support reference types.

```

### Concept-Based Diagnostics

```cpp

#include <concepts>
#include <type_traits>
#include <string>

template<typename T>
concept Serializable =
    requires(const T& obj, std::ostream& os) {
        { obj.serialize(os) } -> std::same_as<void>;
    } &&
    requires {
        { T::type_name() } -> std::convertible_to<std::string>;
    };

// When the concept fails:
// "constraint not satisfied: 'obj.serialize(os)' is not a valid expression"
// Much better than SFINAE's "no matching function"

template<Serializable T>
void save(const T& obj);

// Layered concepts + static_assert:
template<typename T>
void detailed_check() {
    static_assert(requires(const T& obj, std::ostream& os) {
        obj.serialize(os);
    }, "T must have a serialize(ostream&) method");

    static_assert(requires { T::type_name(); },
                  "T must have a static type_name() method");

    static_assert(std::is_nothrow_move_constructible_v<T>,
                  "T must be noexcept movable for container compatibility");
}

```

### Compile-Time Diagnostics with consteval

```cpp

#include <string_view>
#include <algorithm>

consteval bool validate_identifier(std::string_view name) {
    if (name.empty()) return false;
    if (!std::isalpha(name[0]) && name[0] != '_') return false;
    return std::all_of(name.begin(), name.end(),
                       [](char c) { return std::isalnum(c) || c == '_'; });
}

template<size_t N>
struct FieldName {
    char data[N];
    consteval FieldName(const char (&str)[N]) {
        std::copy(str, str + N, data);
        static_assert(validate_identifier({str, N - 1}),
                      "Field name must be a valid identifier");
    }
};

```

---

## Self-Assessment

### Q1: Why are concept error messages better than SFINAE

Concepts report which specific requirement failed: "the expression `obj.serialize(os)` is not valid." SFINAE reports "no matching function" and dumps the entire overload set. Concepts diagnose the WHY; SFINAE only says WHAT failed.

### Q2: Show how to test multiple requirements with individual error messages

Use multiple `static_assert`s, each testing one requirement. The first failure gives a specific message. With a single concept, you get one combined error. With individual assertions, each failure is isolated and named.

### Q3: When to use static_assert vs concepts vs SFINAE

- **static_assert**: hard errors inside a template body — "this type is NOT supported."
- **Concepts**: soft errors in overload resolution — "try another overload."
- **SFINAE**: legacy code, fine-grained overload control before C++20.

---

## Notes

- `static_assert(false)` is ill-formed even in uninstantiated templates (C++23 relaxes this).
- Use `static_assert` in the class body to fail early with clear messages.
- Concept subsumption provides better overload resolution than SFINAE.
- Combine concepts and static_assert: concepts for overload selection, static_assert for hard requirements.
