# Understand C++26 Contracts in depth: assertion levels, build modes, continuation

**Category:** Error Handling  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2900>  

---

## Topic Overview

C++26 Contracts add syntax for preconditions, postconditions, and assertions, enforced by the build system with configurable levels and continuation modes.

### Basic Syntax

```cpp

// Precondition: checked before function body
int sqrt_int(int x)
    pre(x >= 0)   // Contract: x must be non-negative
{
    // ...
}

// Postcondition: checked after function returns
int abs_val(int x)
    post(r: r >= 0)  // Contract: return value r is non-negative
{
    return x < 0 ? -x : x;
}

// Assertion: checked at a specific point
void process(std::vector<int>& v) {
    contract_assert(v.size() > 0);
    // ...
}

```

### Assertion Levels

```cpp

// Three levels control when checks are active:
int divide(int a, int b)
    pre(b != 0)           // Default level
    pre audit(a > 0)      // Audit: expensive checks, off in production
{
    contract_assert(b != 0);          // Default
    contract_assert audit(a > 0);    // Audit level
    return a / b;
}

// Build modes (implementation-defined):
// - Off:      No checks
// - Default:  Default-level checks only
// - Audit:    All checks including audit

```

### Violation Handlers and Continuation

```cpp

// When a contract is violated:
// 1. A violation handler is called with contract_violation info
// 2. Depending on build mode, program may:
//    a) Terminate (default)
//    b) Continue execution (if continuation mode is set)

// The contract_violation structure:
// - line_number, file_name, function_name
// - comment (the text of the condition)
// - assertion_level (default, audit)
// - detection_mode (evaluation or assume)

// Custom violation handler (implementation-defined mechanism):
void handle_violation(const std::contract_violation& v) {
    std::cerr << v.file_name() << ":" << v.line_number()
              << " Contract violated: " << v.comment() << "\n";
    // In "continue" mode, execution continues after this returns
    // In "terminate" mode, std::terminate() is called after this
}

```

---

## Self-Assessment

### Q1: How do contracts differ from assert()

Contracts are part of the function signature (visible to callers and tools). They have standardized levels (default, audit). They can be checked by static analyzers. They have standardized violation handling. `assert()` is a macro with no semantic meaning to the compiler.

### Q2: What is the "assume" evaluation mode

In "assume" mode, contracts are not checked at runtime. Instead, the compiler assumes they are true and optimizes accordingly (`[[assume(expr)]]`). This is dangerous — a violated assumption causes UB — but enables maximum optimization.

### Q3: Show how contracts aid static analysis

```cpp

int access(const std::vector<int>& v, size_t i)
    pre(i < v.size());
// A static analyzer can:
// 1. Check callers provide valid indices
// 2. Assume i < v.size() inside the function (no bounds check needed)
// 3. Warn if precondition might not hold

```

---

## Notes

- C++26 Contracts (P2900) is the biggest C++26 feature.
- Contracts are NOT yet implemented in any compiler (as of 2025).
- The syntax may change before the C++26 standard is finalized.
- Contracts replace many uses of `assert()`, `GSL_EXPECTS()`, and `BOOST_ASSERT()`.
