# Understand placeholder variables with _ and auto (C++26)

**Category:** Core Language Fundamentals  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2169>  

---

## Topic Overview

C++26 allows `_` (underscore) as a placeholder variable name that can be declared multiple times in the same scope. This signals "I don't care about this value."

### Before C++26

Before C++26, ignoring unwanted values from a structured binding was awkward. You had to invent names, and then suppress the warnings those invented names generated:

```cpp
#include <tuple>
#include <iostream>

auto get_data() { return std::tuple{42, 3.14, "hello"}; }

void old_style() {
    auto [x, unused1, unused2] = get_data();  // Must invent names
    // unused1 and unused2 trigger "unused variable" warnings
    (void)unused1; (void)unused2;  // Suppress warnings manually
    std::cout << x << "\n";
}
```

### C++26 Placeholder

Now `_` can appear multiple times in the same scope - the compiler understands that each one is a separate unnamed slot you're deliberately ignoring:

```cpp
#include <tuple>
#include <iostream>

void new_style() {
    auto [x, _, _] = get_data();  // C++26: _ can appear multiple times!
    std::cout << x << "\n";       // Only x is used
    // No warnings, no name invention
}

// Also works in other contexts:
void examples() {
    // Multiple _ in same scope:
    auto _ = some_lock_guard();    // Name doesn't matter
    auto _ = some_other_raii();    // Multiple _ OK!

    // In lambda captures (same rule):
    int a = 1, b = 2;
    auto f = [a, _=std::move(b)]() { return a; };  // _ discards b's name
}
```

### Structured Bindings

The most common use case is structured bindings where you only care about some of the values. Prefer a real name when you actually use the value - use `_` only when you genuinely don't:

```cpp
// Most common use: structured binding placeholders
if (auto [iter, _] = my_map.insert({key, value}); !_) {
    // Note: _ IS the bool, but we can't use it after declaring _ twice
}

// Better:
if (auto [iter, inserted] = my_map.insert({key, value}); inserted) {
    // Use inserted when you need it, _ when you don't
}
```

---

## Self-Assessment

### Q1: Can you read from a _ variable

No. When `_` is declared multiple times in the same scope, each declaration refers to a different unnamed variable and none can be read. If `_` appears only once, it acts as a normal variable with the name `_` and CAN be read.

### Q2: How does this differ from [[maybe_unused]]

`[[maybe_unused]]` suppresses warnings for a named variable you might or might not use. `_` signals you will NEVER use the value. `_` is also shorter and can appear multiple times. The contrast is: `[[maybe_unused]] auto x = f();` versus `auto _ = f();`.

### Q3: What about existing code that uses _ as a variable name

Code that uses `_` as a regular variable name (single occurrence) continues to work unchanged. The new behavior only activates when `_` is declared multiple times in the same scope. Libraries that use `_` for gettext (`_("string")`) are unaffected since that's a function name, not a variable.

---

## Notes

- `_` as a placeholder is borrowed from Python, Rust, Go, and other languages.
- Primary motivation: structured bindings where you only need some values.
- Only variable declarations are affected - `_` as a function/type name is unchanged.
- GCC 14+, Clang 18+ support this feature.
