# Debug template instantiation errors systematically with concept diagnostics

**Category:** Debugging Advanced Techniques  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints>  

---

## Topic Overview

Template error messages are notoriously long and cryptic. Before C++20, a single wrong argument could produce hundreds of lines of substitution failure output where the actual problem is buried somewhere near the bottom. C++20 concepts change that picture dramatically by letting the compiler tell you exactly which constraint failed, and systematic techniques let you tame even pre-C++20 errors when you're stuck with older code.

### Before Concepts (SFINAE Error Soup)

Here is what the pre-concepts world looked like. You call something that expects a range, pass the wrong type, and the compiler buries the real issue under a wall of "substitution failure" and "candidate template ignored" notes:

```cpp
// Error: 250 lines of template instantiation backtrace
// "no matching function for call to 'sort'"
// "candidate template ignored: substitution failure"
// "no member named 'begin' in 'int'"

// The actual problem: you passed an int where a range was expected
std::sort(42, 100);  // Error buried in layers of template instantiation
```

The code is obviously wrong, but you have to read through a wall of diagnostics to find out *why*. This is the classic SFINAE error experience that has frustrated C++ developers for decades.

### With Concepts (Clean Diagnostics)

With C++20 concepts, the compiler knows exactly what the template requires and can say so in one line. The error stops being a trail of breadcrumbs and becomes a direct statement of the problem:

```cpp
#include <concepts>
#include <ranges>

template<std::ranges::random_access_range R>
void my_sort(R&& range);

// my_sort(42);
// Error: constraints not satisfied
// note: 'std::ranges::random_access_range<int>' was not satisfied
// <- ONE LINE tells you exactly what's wrong
```

That single "constraints not satisfied" note replaces the 250-line backtrace. The reason the diagnostic is so much cleaner is that concepts express the requirements at the template declaration level, so the compiler can report the failure there rather than deep inside an instantiation chain.

### Systematic Debugging Technique: Concept Checking

When you do get a confusing template error, the most reliable way to isolate the problem is to add explicit `static_assert` checkpoints that test each requirement one at a time. This technique works even in C++11/14/17 code, long before concepts existed. The idea is to ask the compiler a series of yes/no questions about your type until you find exactly which property is missing:

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

When you call `diagnose<MyType>()`, the first failing `static_assert` tells you exactly which property your type is missing. That is far easier to act on than a generic "no matching function" message.

### Compiler Flags for Better Template Errors

Different compilers also give you flags to tune how much detail they show. GCC tends to produce very long backtraces by default, and you can trim them. Clang can show you a visual diff between the type you provided and the type the template expected:

```bash
# GCC: limit template backtrace depth
g++ -ftemplate-backtrace-limit=5 -fconcepts-diagnostics-depth=2 main.cpp

# Clang: better template error formatting
clang++ -ftemplate-depth=32 -fdiagnostics-show-template-tree main.cpp

# Clang: show template argument differences in overload candidates
# -fdiagnostics-show-template-tree shows AST diff between expected and actual types
```

The `-ftemplate-backtrace-limit=5` flag on GCC is especially useful when you're first triaging an error - it keeps the output small enough to actually read. Once you know which level of the instantiation is breaking, you can increase the limit to see more detail.

---

## Self-Assessment

### Q1: How to decode a long SFINAE error

When you face a wall of template error output, there is a reliable reading strategy that cuts straight to the issue:

1. Start from the bottom of the error - the actual cause is usually the innermost failure.
2. Look for "required from here" to find the user's call site.
3. Look for "substitution failure" to find which overload was rejected and why.
4. Extract the type mismatch: what type was provided vs what was expected.

The reason to start from the bottom is that compilers print the instantiation chain from outermost to innermost. Your call is at the top, and the actual failed operation is at the very end.

### Q2: Show how to create a "static assert checkpoint" for complex templates

This technique is essentially defensive programming for templates. You put the constraints right at the class body entry point, so misuse fails early with a message you wrote, rather than deep inside some implementation detail:

```cpp
template<typename T>
class Container {
    // Checkpoint: fail early with clear message
    static_assert(!std::is_reference_v<T>,
                  "Container<T&> is not supported - use reference_wrapper");
    static_assert(!std::is_void_v<T>,
                  "Container<void> is not supported");
    static_assert(std::is_destructible_v<T>,
                  "T must be destructible");
    // ... rest of class only compiles if checks pass
};
```

The rest of the class body only participates in compilation if all three assertions pass. That means a user who passes `Container<int&>` gets a clear English sentence telling them what to do instead.

### Q3: Use `-ftime-trace` to find which template instantiation is slow

If your build is slow and you suspect heavy template instantiation, Clang can generate a detailed timing breakdown in a format you can visualize directly in Chrome:

```bash
clang++ -ftime-trace main.cpp -o main
# Opens a Chrome trace viewer JSON showing:
# - Which headers took the most parse time
# - Which template instantiations were slowest
# - Total time per compilation phase
```

Load the resulting `.json` file in `chrome://tracing` and you'll see a flame chart where the width of each bar corresponds directly to compile time. The widest bars are your bottlenecks.

---

## Notes

- Always prefer concepts over SFINAE for new code - the diagnostics alone are worth it.
- `static_assert` is the best tool for catching misuse early with clear messages.
- GCC's `-fconcepts-diagnostics-depth=N` controls how deep concept failure explanation goes.
- Clang's `-fdiagnostics-show-template-tree` shows type differences visually.
