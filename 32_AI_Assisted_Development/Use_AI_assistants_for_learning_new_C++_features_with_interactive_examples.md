# Use AI assistants for learning new C++ features with interactive examples

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI assistants are **exceptional learning tools** for new C++ features. They can explain concepts progressively (simple → advanced), generate runnable examples, create exercises with solutions, compare old vs. new approaches, and answer follow-up questions instantly. This creates a personalized, interactive learning experience far more effective than reading documentation alone.

### Effective Learning Prompts

| Learning Goal | Prompt Pattern | Example |
| --- | --- | --- |
| Understand a feature | "Explain X with 3 examples, easy to hard" | "Explain C++20 concepts" |
| Compare old vs new | "Show how X was done before and after Y" | "SFINAE vs concepts" |
| Apply to real code | "Refactor this code to use X" | "Use ranges here" |
| Edge cases | "What are the gotchas of X?" | "std::move gotchas" |
| Build intuition | "When should I use X vs Y?" | "coroutines vs threads" |
| Practice | "Give me 5 exercises on X with solutions" | "constexpr exercises" |

---

## Self-Assessment

### Q1: Learn a new feature progressively

**Answer:**

```cpp

=== PROMPT ===

"Teach me C++20 concepts in 4 progressive steps:

1. What they replace and why (motivation)
2. Basic syntax with simple examples
3. Composing concepts and requires expressions
4. Advanced: subsumption, constrained auto, concept maps

For each step, give a compilable example and an exercise."


=== AI RESPONSE (Step 1: Motivation) ===

"Concepts replace SFINAE-based constraints. Here's why:"

```

```cpp

// BEFORE (C++17 SFINAE): Cryptic error messages
template<typename T,
    typename = std::enable_if_t<
        std::is_integral_v<T> && sizeof(T) >= 4>>
void process(T value) { /* ... */ }

process(3.14);  // Error: no matching function 'process'
// Error message: pages of template instantiation failures

// AFTER (C++20 concepts): Clear constraint
template<typename T>
concept LargeInteger = std::integral<T> && (sizeof(T) >= 4);

void process(LargeInteger auto value) { /* ... */ }

process(3.14);  // Error: constraints not satisfied:
                // 'double' does not satisfy 'LargeInteger'
                // because: double is not integral

// Exercise: Write a concept 'Printable' that requires
// T to have operator<< with std::ostream.

```

```cpp

=== AI RESPONSE (Step 2: Basic syntax) ===

```

```cpp

// Three ways to constrain templates:

// Way 1: requires clause
template<typename T>
    requires std::integral<T>
T double_it(T x) { return x * 2; }

// Way 2: Constrained template parameter
template<std::integral T>
T triple_it(T x) { return x * 3; }

// Way 3: Abbreviated function template
auto quadruple_it(std::integral auto x) { return x * 4; }

// All three are equivalent for single constraints.
// Use Way 1 for complex constraints, Way 3 for simple ones.

// Custom concept:
template<typename T>
concept Numeric = std::integral<T> || std::floating_point<T>;

void compute(Numeric auto x) {
    // Works with int, double, float, etc.
}

// Exercise: Create a concept 'Container' that requires:
// - begin() and end() methods
// - a value_type alias
// - a size() method returning a size_t-like type

```

### Q2: Compare old vs new approaches

**Answer:**

```cpp

=== PROMPT ===

"Show me 5 common C++ patterns that are better in C++20.
For each: old way, new way, and when to still use the old way."

```

```cpp

// === 1. Iteration: for loop vs ranges ===

// Old: manual iterator loop
std::vector<int> v{3,1,4,1,5};
std::vector<int> result;
for (auto it = v.begin(); it != v.end(); ++it) {
    if (*it > 2) result.push_back(*it * *it);
}
std::sort(result.begin(), result.end());

// New: ranges pipeline
auto result = v
    | std::views::filter([](int x) { return x > 2; })
    | std::views::transform([](int x) { return x * x; })
    | std::ranges::to<std::vector>();  // C++23
std::ranges::sort(result);

// Still use old when: custom iterator logic, performance-critical
// inner loops where you need iterator invalidation control.


// === 2. String formatting ===

// Old:
char buf[256];
snprintf(buf, sizeof(buf), "User %s (id=%d)", name, id);  // Unsafe
std::string msg = "User " + name + " (id=" + std::to_string(id) + ")";  // Slow

// New:
auto msg = std::format("User {} (id={})", name, id);  // Safe + fast

// Still use old when: embedded systems without stdlib, hot paths
// where format string parsing overhead matters.


// === 3. Variant visitation ===

// Old:
struct Visitor {
    void operator()(int x) const { /*...*/ }
    void operator()(const std::string& s) const { /*...*/ }
    void operator()(double d) const { /*...*/ }
};
std::visit(Visitor{}, my_variant);

// New: overload pattern (C++20 with CTAD)
template<class... Ts>
struct overloaded : Ts... { using Ts::operator()...; };

std::visit(overloaded{
    [](int x) { /*...*/ },
    [](const std::string& s) { /*...*/ },
    [](double d) { /*...*/ }
}, my_variant);


// === 4. Compile-time branching ===

// Old: tag dispatch or SFINAE
template<typename T>
auto serialize(const T& val)
    -> std::enable_if_t<has_to_string_v<T>, std::string>;

// New: constexpr if + concepts
template<typename T>
std::string serialize(const T& val) {
    if constexpr (requires { val.to_string(); })
        return val.to_string();
    else if constexpr (std::formattable<T, char>)
        return std::format("{}", val);
    else
        static_assert(false, "No serialization for T");
}

```

### Q3: Practice with exercises

**Answer:**

```cpp

=== PROMPT ===

"Give me 3 progressive exercises on C++20 coroutines.
For each: problem statement, hints, and full solution.
Difficulty: beginner, intermediate, advanced."

```

```cpp

// === Exercise 1 (Beginner): Simple generator ===
// Problem: Write a generator that yields Fibonacci numbers.
// Hint: You need a Generator<T> type with promise_type.

// Solution:
#include <coroutine>

template<typename T>
class Generator {
public:
    struct promise_type {
        T current_value;
        Generator get_return_object() {
            return Generator{Handle::from_promise(*this)};
        }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        std::suspend_always yield_value(T value) {
            current_value = value;
            return {};
        }
        void return_void() {}
        void unhandled_exception() { std::terminate(); }
    };

    using Handle = std::coroutine_handle<promise_type>;

    bool next() {
        if (handle_ && !handle_.done()) {
            handle_.resume();
            return !handle_.done();
        }
        return false;
    }
    T value() const { return handle_.promise().current_value; }

    ~Generator() { if (handle_) handle_.destroy(); }
    Generator(Generator&& o) noexcept
        : handle_(std::exchange(o.handle_, nullptr)) {}

private:
    Generator(Handle h) : handle_(h) {}
    Handle handle_;
};

// Fibonacci generator coroutine
Generator<uint64_t> fibonacci() {
    uint64_t a = 0, b = 1;
    while (true) {
        co_yield a;
        auto next = a + b;
        a = b;
        b = next;
    }
}

// Usage:
auto fib = fibonacci();
for (int i = 0; i < 10 && fib.next(); ++i) {
    std::cout << fib.value() << ' ';
}
// Output: 0 1 1 2 3 5 8 13 21 34

```

---

## Notes

- **"Explain like I know C++17 but not C++20"** is an effective prompt for learning new features
- Ask for **compilable examples** — AI sometimes generates code that doesn't actually compile
- Use AI for **"what's new in C++23"** summaries with code examples for each feature
- **Follow-up questions** are powerful: "What happens if I remove the `const`?" "Why is this `noexcept`?"
- Ask AI to **generate Compiler Explorer links** for trying code instantly
- AI is excellent at **explaining error messages** from new features like concepts and coroutines
- For deep learning, ask AI to **explain the standard wording** for a specific feature
