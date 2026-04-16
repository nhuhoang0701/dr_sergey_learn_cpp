# Debug template instantiation errors systematically with concept diagnostics

**Category:** Debugging Advanced Techniques  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints>  

---

## Topic Overview

Template error messages are notoriously long and cryptic. C++20 concepts dramatically improve diagnostics, and systematic techniques tame even pre-C++20 errors.

### Before Concepts (SFINAE Error Soup)

```cpp

// Error: 250 lines of template instantiation backtrace
// "no matching function for call to 'sort'"
// "candidate template ignored: substitution failure"
// "no member named 'begin' in 'int'"

// The actual problem: you passed an int where a range was expected
std::sort(42, 100);  // Error buried in layers of template instantiation

```

### With Concepts (Clean Diagnostics)

```cpp

#include <concepts>
#include <ranges>

template<std::ranges::random_access_range R>
void my_sort(R&& range);

// my_sort(42);
// Error: constraints not satisfied
// note: 'std::ranges::random_access_range<int>' was not satisfied
// ← ONE LINE tells you exactly what's wrong

```

### Systematic Debugging Technique: Concept Checking

```cpp

#include <concepts>
#include <type_traits>
#include <vector>

// When you get a template error, check concepts step by step:
template<typename T>
void diagnose() {
    // Check each requirement individually
    static_assert(std::is_default_constructible_v<T>,
                  "T must be default constructible");
    static_assert(std::is_copy_constructible_v<T>,
                  "T must be copy constructible");
    static_assert(std::is_move_constructible_v<T>,
                  "T must be move constructible");

    // Check specific operations
    static_assert(requires(T a, T b) { a < b; },
                  "T must support operator<");
    static_assert(requires(T a, T b) { a == b; },
                  "T must support operator==");
}

// Usage when debugging: diagnose<MyType>();  // Fails with specific message

```

### Compiler Flags for Better Template Errors

```bash

# GCC: limit template backtrace depth
g++ -ftemplate-backtrace-limit=5 -fconcepts-diagnostics-depth=2 main.cpp

# Clang: better template error formatting
clang++ -ftemplate-depth=32 -fdiagnostics-show-template-tree main.cpp

# Clang: show template argument differences in overload candidates
# -fdiagnostics-show-template-tree shows AST diff between expected and actual types

```

---

## Self-Assessment

### Q1: How to decode a long SFINAE error

1. Start from the bottom of the error — the actual cause is usually the innermost failure.
2. Look for "required from here" to find the user's call site.
3. Look for "substitution failure" to find which overload was rejected and why.
4. Extract the type mismatch: what type was provided vs what was expected.

### Q2: Show how to create a "static assert checkpoint" for complex templates

```cpp

template<typename T>
class Container {
    // Checkpoint: fail early with clear message
    static_assert(!std::is_reference_v<T>,
                  "Container<T&> is not supported — use reference_wrapper");
    static_assert(!std::is_void_v<T>,
                  "Container<void> is not supported");
    static_assert(std::is_destructible_v<T>,
                  "T must be destructible");
    // ... rest of class only compiles if checks pass
};

```

### Q3: Use `-ftime-trace` to find which template instantiation is slow

```bash

clang++ -ftime-trace main.cpp -o main
# Opens a Chrome trace viewer JSON showing:
# - Which headers took the most parse time
# - Which template instantiations were slowest
# - Total time per compilation phase

```

---

## Notes

- Always prefer concepts over SFINAE for new code — the diagnostics alone are worth it.
- `static_assert` is the best tool for catching misuse early with clear messages.
- GCC's `-fconcepts-diagnostics-depth=N` controls how deep concept failure explanation goes.
- Clang's `-fdiagnostics-show-template-tree` shows type differences visually.
