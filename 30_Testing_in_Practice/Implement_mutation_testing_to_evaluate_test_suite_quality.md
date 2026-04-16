# Implement mutation testing to evaluate test suite quality

**Category:** Testing in Practice

---

## Topic Overview

**Mutation testing** evaluates test suite quality by introducing small changes (mutations) to the source code and checking if tests detect them. If a mutant survives (tests still pass), it means the test suite has a gap. The **mutation score** (killed/total mutants) is a more meaningful quality metric than code coverage.

### Mutation Testing vs Code Coverage

| Metric | Code Coverage | Mutation Score |
| --- | --- | --- |
| **Measures** | Which lines are executed | Whether tests detect changes |
| **False confidence** | High — 100% coverage doesn't mean good tests | Low — high scores indicate real verification |
| **Cost** | Cheap (single test run) | Expensive (many test runs) |
| **Typical target** | 80-90% line coverage | 70-80% mutation score is excellent |
| **Limitations** | Doesn't verify assertions exist | Slow, requires many test runs |

### Common Mutation Operators

| Operator | Original | Mutated |
| --- | --- | --- |
| **Arithmetic** | `a + b` | `a - b`, `a * b` |
| **Relational** | `a < b` | `a <= b`, `a > b`, `a == b` |
| **Logical** | `a && b` | `a \|\| b` |
| **Unary** | `-a` | `a` |
| **Return** | `return x;` | `return 0;`, `return x+1;` |
| **Void call** | `do_thing();` | (removed) |
| **Constant** | `42` | `0`, `1`, `43` |

---

## Self-Assessment

### Q1: Set up mutation testing with Mull for C++

**Answer:**

```bash

# === Install Mull (LLVM-based mutation testing) ===
# Ubuntu/Debian
sudo apt-get install mull-18  # Matches your LLVM version

# Verify
mull-runner-18 --version

```

```cmake

# === CMakeLists.txt - Prepare for mutation testing ===
# Mull works on LLVM bitcode, so we need Clang and special flags

option(ENABLE_MULL "Enable mutation testing with Mull" OFF)

if(ENABLE_MULL)
    # Mull requires specific Clang flags
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} \
        -fpass-plugin=/usr/lib/mull-ir-frontend-18 \
        -g -grecord-command-line")
endif()

add_library(my_math src/math_utils.cpp)
add_executable(math_tests tests/test_math.cpp)
target_link_libraries(math_tests my_math GTest::gtest_main)

```

```cpp

// === src/math_utils.h ===
#pragma once

int clamp(int value, int min, int max);
bool is_in_range(int value, int low, int high);
int safe_divide(int a, int b);
double calculate_discount(double price, int quantity);

```

```cpp

// === src/math_utils.cpp ===
#include "math_utils.h"
#include <stdexcept>

int clamp(int value, int min, int max) {
    if (value < min) return min;
    if (value > max) return max;
    return value;
}

bool is_in_range(int value, int low, int high) {
    return value >= low && value <= high;
}

int safe_divide(int a, int b) {
    if (b == 0) throw std::invalid_argument("Division by zero");
    return a / b;
}

double calculate_discount(double price, int quantity) {
    if (quantity >= 100) return price * 0.8;   // 20% off
    if (quantity >= 10)  return price * 0.9;   // 10% off
    return price;
}

```

### Q2: Write tests that survive mutation testing

**Answer:**

```cpp

// === tests/test_math.cpp ===
#include <gtest/gtest.h>
#include "math_utils.h"

// === WEAK tests: these let mutants survive ===
TEST(ClampWeak, BasicTest) {
    EXPECT_EQ(clamp(5, 0, 10), 5);  // Only tests middle path
    // Mutants that survive:
    // - change `value < min` to `value <= min` (not caught!)
    // - change `value > max` to `value >= max` (not caught!)
}

// === STRONG tests: kill all mutants ===
TEST(ClampStrong, ValueInRange) {
    EXPECT_EQ(clamp(5, 0, 10), 5);
}

TEST(ClampStrong, ValueBelowMin) {
    EXPECT_EQ(clamp(-5, 0, 10), 0);
}

TEST(ClampStrong, ValueAboveMax) {
    EXPECT_EQ(clamp(15, 0, 10), 10);
}

// Boundary tests kill `<` vs `<=` mutants
TEST(ClampStrong, ValueExactlyAtMin) {
    EXPECT_EQ(clamp(0, 0, 10), 0);  // Kills `<` -> `<=` mutant
}

TEST(ClampStrong, ValueExactlyAtMax) {
    EXPECT_EQ(clamp(10, 0, 10), 10);  // Kills `>` -> `>=` mutant
}

TEST(ClampStrong, ValueOneAboveMin) {
    EXPECT_EQ(clamp(1, 0, 10), 1);  // Kills off-by-one mutants
}

// === Discount tests: kill arithmetic mutants ===
TEST(DiscountStrong, NoDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 5), 100.0);
}

TEST(DiscountStrong, SmallDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 10), 90.0);
}

TEST(DiscountStrong, LargeDiscount) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 100), 80.0);
}

// Boundary kills `>=` vs `>` mutants
TEST(DiscountStrong, BoundaryAt10) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 9), 100.0);
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 10), 90.0);
}

TEST(DiscountStrong, BoundaryAt100) {
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 99), 90.0);
    EXPECT_DOUBLE_EQ(calculate_discount(100.0, 100), 80.0);
}

// === Range tests ===
TEST(IsInRangeStrong, Inside) {
    EXPECT_TRUE(is_in_range(5, 0, 10));
}

TEST(IsInRangeStrong, AtBoundaries) {
    EXPECT_TRUE(is_in_range(0, 0, 10));   // Kills >= vs > mutant
    EXPECT_TRUE(is_in_range(10, 0, 10));  // Kills <= vs < mutant
}

TEST(IsInRangeStrong, JustOutside) {
    EXPECT_FALSE(is_in_range(-1, 0, 10));  // Kills boundary mutants
    EXPECT_FALSE(is_in_range(11, 0, 10));
}

TEST(IsInRangeStrong, BothConditions) {
    // Kills `&&` -> `||` mutant: value outside one bound
    EXPECT_FALSE(is_in_range(-1, 0, 10));
    EXPECT_FALSE(is_in_range(11, 0, 10));
}

```

### Q3: Run and interpret mutation testing results

**Answer:**

```bash

# === Build with Mull instrumentation ===
cmake -B build-mull \
    -DCMAKE_CXX_COMPILER=clang++-18 \
    -DENABLE_MULL=ON \
    -DCMAKE_BUILD_TYPE=Debug
cmake --build build-mull

# === Run mutation testing ===
mull-runner-18 ./build-mull/math_tests \
    --reporters=Elements \
    --report-dir=mull-report

# === Example output ===
# [info] Applying 47 mutations
# [info] Running mutants (47 total):
# Killed:    41 (87.2%)
# Survived:   4 (8.5%)
# Timeout:    2 (4.3%)
#
# Survived mutants:
#   src/math_utils.cpp:15: Replaced >= with > in is_in_range
#   src/math_utils.cpp:22: Replaced * with / in calculate_discount

# === mull.yml configuration ===
# Place in project root

```

```yaml

# === mull.yml ===
mutators:

  - cxx_arithmetic         # +, -, *, /
  - cxx_comparison         # <, <=, >, >=, ==, !=
  - cxx_logical            # &&, ||
  - cxx_remove_void_call   # Remove void function calls
  - cxx_replace_scalar     # Replace constants

excludePaths:

  - "third_party/.*"       # Don't mutate dependencies
  - "tests/.*"             # Don't mutate test code
  - ".*_generated\.cpp"    # Don't mutate generated code

timeout: 10000             # 10 seconds per mutant

```

---

## Notes

- **Mutation score > code coverage** for test quality assessment — it proves tests actually verify behavior
- Equivalent mutants (mutations that don't change behavior) inflate the denominator — expect ~5-10%
- Start mutation testing on critical/complex modules, not the entire codebase
- Boundary tests are the #1 way to kill relational operator mutants
- If a mutant survives, ask: "What assertion would catch this change?" and add it
- Mull works on LLVM IR, so it requires Clang. GCC users can try `mutate_cpp` or `dextool`
- Mutation testing is slow (N test runs for N mutants) — run nightly or on critical paths only
- Target 70-80% mutation score for critical code; 100% is impractical due to equivalent mutants
