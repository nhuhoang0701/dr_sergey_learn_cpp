# Use contracts (C++26 preview) and precondition/postcondition annotations

**Category:** Error Handling  
**Item:** #166  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/language/contracts>  

---

## Topic Overview

**C++26 contracts** introduce standardized syntax for **preconditions**, **postconditions**, and **assertions** directly in function declarations. Unlike `assert()`, contracts are part of the type system, visible in declarations, and have configurable runtime behavior.

### Contract Syntax (C++26)

```cpp

// Precondition: checked before function body executes
int sqrt_int(int x)
    pre(x >= 0)           // precondition
    post(r: r >= 0)       // postcondition: 'r' names the return value
{
    contract_assert(x < 1000000);  // assertion within function body
    return /* implementation */;
}

// Multiple conditions
void copy(const char* src, char* dst, size_t n)
    pre(src != nullptr)
    pre(dst != nullptr)
    pre(n > 0)
    post(dst[0] == src[0])  // first byte copied correctly
{
    // ...
}

```

### Three Contract Annotation Types

| Annotation | Where | Checks |
| --- | --- | --- |
| `pre(expr)` | Function declaration | Caller's obligation (before call) |
| `post(r: expr)` | Function declaration | Function's guarantee (after return) |
| `contract_assert(expr)` | Function body | Invariant within function |

### Contract Violation Modes

The contract evaluation semantic is set globally at build time (not per-contract):

| Mode | Behavior on Violation | Use Case |
| --- | --- | --- |
| **ignore** | No check generated (contracts removed entirely) | Maximum performance (release build) |
| **observe** | Check runs; on violation, calls handler but **continues** | Gradual adoption / logging |
| **enforce** | Check runs; on violation, calls handler and **does not continue** | Debug / testing |

```cpp

Build modes:
  -fcontract-semantic=ignore    →  zero runtime cost (like NDEBUG for assert)
  -fcontract-semantic=observe   →  log violations but don't crash
  -fcontract-semantic=enforce   →  crash on violation (like assert)

```

### Important Notes

- Contracts are evaluated in the **caller's context** (for `pre`/`post`) — they can be part of the ABI.
- The `post(r: expr)` syntax names the return value as `r` (or any identifier).
- Contract conditions must not have side effects (implementation may evaluate them zero or more times).
- C++26 contracts are **not yet available** in any major compiler (as of 2024). Use `assert()` or custom macros as a bridge.

---

## Self-Assessment

### Q1: Write a function with `pre(x > 0)` and `post(result >= 0)` annotations

**Solution — C++26 Contracts Syntax:**

```cpp

#include <iostream>
#include <cmath>
#include <cassert>

// ========== C++26 contracts (when available) ==========

// Square root with contracts
double safe_sqrt(double x)
    pre(x >= 0.0)             // caller must ensure non-negative input
    post(r: r >= 0.0)         // function guarantees non-negative result
    post(r: std::abs(r * r - x) < 0.0001)  // result is approximately correct
{
    return std::sqrt(x);
}

// Binary search with contracts
int binary_search(const int* arr, int n, int target)
    pre(arr != nullptr)       // non-null array
    pre(n > 0)                // non-empty
    // pre: arr is sorted (can't easily express in a single expression)
    post(r: r == -1 || (r >= 0 && r < n))  // valid index or -1
{
    int lo = 0, hi = n - 1;
    while (lo <= hi) {
        int mid = lo + (hi - lo) / 2;
        if (arr[mid] == target) return mid;
        if (arr[mid] < target) lo = mid + 1;
        else hi = mid - 1;
    }
    return -1;
}

// Division with multiple preconditions
double safe_divide(double a, double b)
    pre(b != 0.0)                    // no division by zero
    pre(!std::isnan(a))              // valid inputs
    pre(!std::isnan(b))
    post(r: !std::isnan(r))          // valid output
{
    return a / b;
}

// ========== Today's equivalent with assert() ==========
// (Use until C++26 compilers ship)

double safe_sqrt_today(double x) {
    assert(x >= 0.0 && "Precondition: x must be non-negative");
    double result = std::sqrt(x);
    assert(result >= 0.0 && "Postcondition: result must be non-negative");
    return result;
}

int main() {
    std::cout << safe_sqrt_today(25.0) << "\n";   // 5
    std::cout << safe_sqrt_today(0.0) << "\n";    // 0

    // This would trigger the precondition:
    // std::cout << safe_sqrt_today(-1.0) << "\n";  // assertion failure!
}
// Expected output:
//   5
//   0

```

**Key Syntax Details:**

- `pre(expression)` — evaluated before the function body.
- `post(r: expression)` — `r` is the return value; evaluated after the function returns.
- `contract_assert(expression)` — checked at that point in the function body.
- Multiple `pre`/`post` clauses are allowed — all are checked.

---

### Q2: Explain the three contract violation modes: ignore, observe, enforce

**Complete Explanation:**

```cpp

Contract evaluation semantics:

┌──────────────────────────────────────────────────────────────────┐
│                    Contract Check Point                          │
│                   pre(x > 0)                                     │
│                        │                                         │
│                   Is mode "ignore"?                               │
│                   ┌────┴────┐                                    │
│                  YES        NO                                   │
│                   │         │                                    │
│              (no check)   Evaluate expression                    │
│                   │         │                                    │
│              Continue    Expression true?                         │
│              execution   ┌────┴────┐                             │
│                         YES        NO (violation!)               │
│                          │         │                             │
│                     Continue    Is mode "observe"?               │
│                     execution   ┌────┴────┐                     │
│                                YES        NO (enforce)          │
│                                 │         │                     │
│                            Call handler   Call handler           │
│                            Continue!      DO NOT continue        │
│                                           (terminate)            │
└──────────────────────────────────────────────────────────────────┘

```

**Detailed Comparison:**

| Aspect | `ignore` | `observe` | `enforce` |
| --- | --- | --- | --- |
| Expression evaluated? | No | Yes | Yes |
| On violation: handler called? | No | Yes | Yes |
| On violation: continues? | N/A | **Yes** (handler returns) | **No** (handler must not return) |
| Runtime cost | Zero | Check cost + handler cost | Check cost |
| Use case | Shipped release builds | Migration / logging | Development / testing |
| Analogy | `NDEBUG` + `assert()` | Logging middleware | `assert()` in debug |

**Migration Strategy:**

```cpp

Phase 1: Add contracts in "observe" mode
  → Find all violations in production logs
  → Fix them without breaking existing behavior
  → No crashes — just logging

Phase 2: Switch to "enforce" mode in CI/testing
  → All tests must pass with contracts enforced
  → Catches bugs early in development

Phase 3: Ship with "ignore" or "enforce" per preference
  → "ignore": maximum performance
  → "enforce": maximum safety

```

**Custom Contract Violation Handler (C++26):**

```cpp

// The default handler calls std::terminate().
// You can install a custom handler:

void my_contract_handler(const std::contract_violation& v) {
    std::cerr << "Contract violated!\n"
              << "  Kind: " << (v.kind() == std::contract_kind::pre ? "pre" :
                                v.kind() == std::contract_kind::post ? "post" : "assert")
              << "\n"
              << "  Location: " << v.location().file_name()
              << ":" << v.location().line() << "\n"
              << "  Comment: " << v.comment() << "\n";
    // In "observe" mode, returning from here continues execution
    // In "enforce" mode, returning has undefined behavior
}

```

---

### Q3: Compare contracts with `assert()` and explain the advantages for production builds

**Head-to-Head Comparison:**

| Feature | `assert()` | C++26 Contracts |
| --- | --- | --- |
| **Syntax location** | Inside function body | On function **declaration** |
| **Visible to caller?** | No (implementation detail) | **Yes** — part of the interface |
| **Postconditions?** | Manual (check before return) | Built-in `post(r: expr)` |
| **Build modes** | Binary: on/off (`NDEBUG`) | Three: ignore/observe/enforce |
| **Gradual adoption** | No "log but continue" mode | **observe** mode enables logging |
| **Side effects** | Expression may have side effects | Must be side-effect-free |
| **Optimizer hints** | None | **ignore** mode may use as `[[assume]]` |
| **Tooling** | No special support | IDE/analyzer can read contracts from headers |
| **Virtual functions** | N/A | Contracts on base apply to overrides |
| **Standard?** | C-era macro | C++26 language feature |

**The Key Advantages for Production:**

```cpp

// 1. OBSERVE mode: log violations WITHOUT crashing
//    assert():     NDEBUG removes all checks → silent bugs in production
//    Contracts:    observe mode LOGS the violation and continues

// 2. Optimizer hints: in "ignore" mode, the compiler can ASSUME the
//    contract holds → better optimization than assert+NDEBUG
double fast_sqrt(double x)
    pre(x >= 0.0)   // in "ignore" mode: optimizer assumes x >= 0
{
    return __builtin_sqrt(x);  // no need for NaN check
}

// 3. Declaration-level visibility: contracts are API documentation
//    that the compiler can verify
// In header file:
int find(const std::vector<int>& v, int target)
    pre(!v.empty())            // ← caller sees this in the declaration
    post(r: r == -1 || r < static_cast<int>(v.size()));

// 4. Virtual function contracts: base class contracts apply to overrides
class Shape {
public:
    virtual double area() const
        post(r: r >= 0.0) = 0;  // all implementations must return non-negative
};

```

**Today's Bridge: Custom Macro:**

```cpp

// Until C++26 is available, use a custom macro that
// mimics the three-mode behavior:

#if defined(CONTRACT_OBSERVE)
#define PRE(expr) \
    if (!(expr)) { \
        std::cerr << "PRE violation: " #expr " at " \
                  << __FILE__ << ":" << __LINE__ << "\n"; \
    }
#elif defined(CONTRACT_ENFORCE)
#define PRE(expr) \
    if (!(expr)) { \
        std::cerr << "PRE violation: " #expr " at " \
                  << __FILE__ << ":" << __LINE__ << "\n"; \
        std::abort(); \
    }
#else  // CONTRACT_IGNORE
#define PRE(expr) ((void)0)
#endif

double safe_sqrt(double x) {
    PRE(x >= 0.0);
    return std::sqrt(x);
}

```

---

## Notes

- **C++26 contracts are not yet implemented** by GCC, Clang, or MSVC (as of 2024). The feature was voted into C++26 working draft (P2900).
- **Contracts replace P0542/P1429** — earlier contract proposals that were removed from C++20 at the last minute.
- **`[[assume(expr)]]`** (C++23) is a related but different feature — it tells the optimizer to assume `expr` is true (UB if false). Contracts are checked; assumes are not.
- **GSL `Expects()`/`Ensures()`** (Microsoft Guidelines Support Library) are widely used as contract-like macros today.
- **Contract conditions should be fast** — avoid O(n) checks in preconditions of O(1) functions. Expensive checks like "array is sorted" should be debug-only.
- **Thread safety:** Contract evaluation happens in the calling thread's context, before/after the function's body executes.
