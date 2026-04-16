# Test safety-critical C++ code for MISRA and AUTOSAR compliance

**Category:** Testing in Practice

---

## Topic Overview

**MISRA C++** and **AUTOSAR C++14** are coding standards for safety-critical systems (automotive, medical, aerospace). They restrict language features to prevent undefined behavior, improve code predictability, and enable formal verification. Testing compliance requires static analysis, code review, and documented deviations.

### Standard Comparison

| Standard | Base Language | Rules | Focus | Industry |
| --- | --- | --- | --- | --- |
| **MISRA C++:2023** | C++17 | ~180 | Safety, reliability | Automotive, aerospace |
| **AUTOSAR C++14** | C++14 | ~350 | Safety + OOP guidelines | Automotive (AUTOSAR) |
| **CERT C++** | C++17 | ~100 | Security | All industries |
| **JSF AV C++** | C++03 | ~220 | Safety | Aerospace (Lockheed) |
| **SEI CERT** | Latest | ~100 | Security | All |

### Key MISRA Restrictions

| Category | Restriction | Rationale |
| --- | --- | --- |
| **Dynamic memory** | No `new`/`delete` after init | Fragmentation, OOM in real-time |
| **Exceptions** | Restricted throw/catch | Stack unwinding overhead |
| **RTTI** | No `dynamic_cast`, `typeid` | Runtime overhead |
| **Unions** | Restricted use | Type confusion |
| **Recursion** | Prohibited | Stack overflow risk |
| **goto** | Prohibited | Unstructured control flow |
| **Implicit conversions** | Restricted | Unexpected behavior |

---

## Self-Assessment

### Q1: Configure static analysis for MISRA compliance checking

**Answer:**

```bash

# === cppcheck with MISRA addon ===
cppcheck --addon=misra \
    --suppress=missingIncludeSystem \
    --std=c++17 \
    --error-exitcode=1 \
    --xml --xml-version=2 \
    src/ 2> misra_report.xml

# With rule text file (MISRA rules are copyrighted)
cppcheck --addon=misra.json src/
# misra.json:
# {
#   "script": "misra.py",
#   "args": ["--rule-texts=misra_rules.txt"]
# }

```

```yaml

# === .clang-tidy for safety-oriented checks ===
---
Checks: >
  -*,
  bugprone-*,
  cert-*,
  cppcoreguidelines-*,
  misc-no-recursion,
  misc-throw-by-value-catch-by-reference,
  performance-*,
  readability-*,
  -cppcoreguidelines-pro-type-reinterpret-cast,
  -cppcoreguidelines-pro-bounds-pointer-arithmetic

WarningsAsErrors: >
  bugprone-*,
  cert-*

CheckOptions:

  - key: misc-no-recursion

    value: 'true'
...

```

```cmake

# === CMake: Safety-critical build configuration ===
function(add_safety_critical_target name)
    add_library(${name} ${ARGN})

    # Strict compiler warnings
    target_compile_options(${name} PRIVATE
        -Wall -Wextra -Werror
        -Wpedantic
        -Wconversion          # Implicit conversion warnings
        -Wsign-conversion     # Signed/unsigned conversion
        -Wfloat-equal         # Float equality comparison
        -Wshadow              # Variable shadowing
        -Wcast-align          # Cast alignment issues
        -Wnull-dereference    # Null pointer dereference
        -Wdouble-promotion    # float -> double promotion
        -Wformat=2            # Format string issues
        -Wno-unused-parameter # Allow unused params in interfaces
    )

    # Disable features restricted by MISRA
    target_compile_options(${name} PRIVATE
        -fno-exceptions       # MISRA: restricted
        -fno-rtti             # MISRA: no dynamic_cast/typeid
        -fno-unwind-tables    # No stack unwinding
    )

    # Static analysis
    set_target_properties(${name} PROPERTIES
        CXX_CLANG_TIDY "clang-tidy;--config-file=${CMAKE_SOURCE_DIR}/.clang-tidy-safety"
    )
endfunction()

```

### Q2: Write MISRA-compliant C++ code patterns

**Answer:**

```cpp

// === MISRA-compliant patterns ===

#include <cstdint>
#include <array>
#include <algorithm>

// Rule: Use fixed-width integer types
void process_sensor(uint16_t raw_value) {
    // GOOD: explicit types, no implicit narrowing
    uint32_t scaled = static_cast<uint32_t>(raw_value) * 100U;

    // BAD (MISRA violation): implicit conversion
    // int x = raw_value;  // Widening, but implicit

    // GOOD: explicit cast
    int32_t signed_val = static_cast<int32_t>(raw_value);
}

// Rule: No dynamic allocation after initialization
class SensorManager {
public:
    // GOOD: fixed capacity, no heap
    static constexpr size_t MAX_SENSORS = 16;

    bool register_sensor(uint8_t id) {
        if (count_ >= MAX_SENSORS) return false;
        sensors_[count_++] = id;
        return true;
    }

    // BAD: dynamic allocation
    // std::vector<uint8_t> sensors_;  // Uses heap!

private:
    std::array<uint8_t, MAX_SENSORS> sensors_{};
    size_t count_ = 0;
};

// Rule: No recursion
// BAD:
// uint32_t factorial(uint32_t n) {
//     return n <= 1 ? 1 : n * factorial(n - 1);
// }

// GOOD: iterative
uint32_t factorial(uint32_t n) {
    uint32_t result = 1;
    for (uint32_t i = 2; i <= n; ++i) {
        result *= i;
    }
    return result;
}

// Rule: switch must have default, all enums handled
enum class State : uint8_t { Idle, Running, Error };

void handle_state(State s) {
    switch (s) {
        case State::Idle:    /* ... */ break;
        case State::Running: /* ... */ break;
        case State::Error:   /* ... */ break;
        // default: unreachable, but MISRA requires it
        default: break;
    }
}

// Rule: use strong types, not raw integers
struct Milliseconds { uint32_t value; };
struct Meters { float value; };

void set_timeout(Milliseconds ms);  // GOOD: can't pass meters
// void set_timeout(uint32_t ms);   // BAD: could be anything

```

### Q3: Document and test MISRA deviations

**Answer:**

```cpp

// === Deviation documentation template ===
// MISRA C++:2023 Rule 21.6.1: "Dynamic memory shall not be used"
// Deviation ID: DEV-MEM-001
// Justification: Heap allocation is used only during system initialization
//   (first 100ms after boot). After init phase, a flag prevents further
//   allocations. Verified by runtime assertion and CI test.
// Risk assessment: Low — allocation failure during init triggers safe shutdown.
// Approval: Safety Lead, 2024-01-15

// === Runtime enforcement of allocation phase ===
class AllocationGuard {
public:
    static void allow_allocations() { allowed_ = true; }
    static void freeze_allocations() { allowed_ = false; }
    static bool is_allowed() { return allowed_; }

private:
    static inline bool allowed_ = true;
};

// Override new to enforce allocation phase
void* operator new(std::size_t size) {
    if (!AllocationGuard::is_allowed()) {
        // In production: trigger safe shutdown
        // In tests: assertion failure
        assert(false && "Heap allocation after init phase!");
    }
    return std::malloc(size);
}

```

```cpp

// === Tests for MISRA compliance ===
#include <gtest/gtest.h>

TEST(MisraComplianceTest, NoHeapAfterInit) {
    AllocationGuard::allow_allocations();
    // Simulate init phase
    auto* buffer = new uint8_t[1024];  // OK during init
    delete[] buffer;

    AllocationGuard::freeze_allocations();

    // Verify no allocations in operational phase
    // In test: this would trigger assert failure
    EXPECT_FALSE(AllocationGuard::is_allowed());
}

TEST(MisraComplianceTest, FixedContainerNeverAllocates) {
    AllocationGuard::freeze_allocations();

    SensorManager mgr;
    for (int i = 0; i < 16; ++i) {
        EXPECT_TRUE(mgr.register_sensor(i));
    }
    EXPECT_FALSE(mgr.register_sensor(16));  // Full, but no crash/alloc
}

```

```yaml

# === CI: MISRA compliance gate ===
name: Safety Compliance
on: [push, pull_request]
jobs:
  misra:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Install tools

        run: sudo apt-get install -y cppcheck clang-tidy

      - name: MISRA check (cppcheck)

        run: |
          cppcheck --addon=misra \
              --error-exitcode=1 \
              --suppress=missingIncludeSystem \
              src/safety_critical/

      - name: Safety-oriented clang-tidy

        run: |
          cmake -B build -DCMAKE_EXPORT_COMPILE_COMMANDS=ON
          find src/safety_critical -name '*.cpp' | \
              xargs clang-tidy -p build \
                  --config-file=.clang-tidy-safety \
                  --warnings-as-errors='*'

      - name: Verify no recursion

        run: |
          # misc-no-recursion catches all recursive functions
          clang-tidy -p build \
              --checks='-*,misc-no-recursion' \
              --warnings-as-errors='*' \
              src/safety_critical/*.cpp

```

---

## Notes

- MISRA rules are copyrighted — you need a license to use the rule texts in tools
- Use cppcheck's MISRA addon for automated rule checking; supplement with clang-tidy
- **Every deviation must be documented**: rule ID, justification, risk assessment, approval
- `-fno-exceptions -fno-rtti` at the compiler level enforces key MISRA restrictions
- Fixed-width types (`uint32_t`) prevent platform-dependent behavior
- Strong typing (wrapper structs) prevents parameter confusion at API boundaries
- AUTOSAR C++14 is more permissive than MISRA — allows some dynamic allocation with constraints
- Compliance is a process: static analysis + code review + documentation + CI gates
