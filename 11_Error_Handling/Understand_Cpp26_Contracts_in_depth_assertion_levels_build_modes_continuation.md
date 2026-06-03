# Understand C++26 Contracts in depth: assertion levels, build modes, continuation

**Category:** Error Handling  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2900>  

---

## Topic Overview

C++26 Contracts add syntax for preconditions, postconditions, and assertions directly in your function signatures and bodies. The build system then controls which checks are active and what happens when they fail. This is a big step up from plain `assert()` - these annotations become part of the function's interface, not just a hidden implementation detail.

### Basic Syntax

Here is the full contract vocabulary at a glance. There are three kinds: `pre` before the body, `post` after the return, and `contract_assert` inside the body.

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

The syntax feels a bit different from what you are used to, but the mental model is simple: `pre` is the caller's responsibility, `post` is the function's promise, and `contract_assert` is an in-body sanity check.

### Assertion Levels

Not every check belongs in production code. Some checks - like verifying that a large data structure is fully sorted - are fine for testing but too expensive to run on every call. The `audit` level lets you tag those expensive checks separately so the build system can turn them on and off independently.

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

Think of `default` level as the lightweight checks you are always willing to pay for, and `audit` level as the thorough but expensive checks you reserve for debug and testing builds.

### Violation Handlers and Continuation

When a contract is violated, C++26 does not just crash silently. It calls a handler first - and depending on the build mode, execution may either stop or actually continue past the violation. That "continue" option is unusual and deserves special attention: it lets you deploy contracts in "observe" mode on production systems to log violations before committing to hard enforcement.

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

The violation object gives you everything you need for useful diagnostics: the file, line, the text of the failing condition, and which level it was. This is significantly richer than a plain `assert()` failure.

---

## Self-Assessment

### Q1: How do contracts differ from assert()

Contracts are part of the function signature - visible to callers, IDEs, and static analysis tools. They have standardized levels (default, audit). They interact with the optimizer (the "assume" mode can feed proven contracts as hints). They have standardized violation handling with structured information.

`assert()` is a C-era macro with no semantic meaning to the compiler. It is either on or off via `NDEBUG`, it lives inside the function body (invisible at the call site), it gives only file and line, and the compiler cannot use it for optimization hints.

### Q2: What is the "assume" evaluation mode

In "assume" mode, contracts are not checked at runtime. Instead, the compiler assumes they are true and uses that fact during optimization - equivalent to writing `[[assume(expr)]]`. This is dangerous: a violated assumption causes undefined behavior, not a graceful error. Use it only when you have already validated the contracts in a lower build tier and you want maximum performance in the final release.

### Q3: Show how contracts aid static analysis

Contracts express intent in a machine-readable form that static analyzers can reason about. Here is what a tool can do with even a single `pre`:

```cpp
int access(const std::vector<int>& v, size_t i)
    pre(i < v.size());
// A static analyzer can:
// 1. Check callers provide valid indices
// 2. Assume i < v.size() inside the function (no bounds check needed)
// 3. Warn if precondition might not hold
```

Without contracts, the analyzer has to guess your intent. With contracts, you have told it exactly what must be true, and it can flag every call site that might violate that promise.

---

## Notes

- C++26 Contracts (P2900) is the biggest C++26 feature.
- Contracts are NOT yet implemented in any compiler (as of 2025).
- The syntax may change before the C++26 standard is finalized.
- Contracts replace many uses of `assert()`, `GSL_EXPECTS()`, and `BOOST_ASSERT()`.
