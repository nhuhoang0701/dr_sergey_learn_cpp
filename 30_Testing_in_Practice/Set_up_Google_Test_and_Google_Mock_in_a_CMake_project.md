# Set up Google Test and Google Mock in a CMake project

**Category:** Testing in Practice

---

## Topic Overview

Google Test (gtest) and Google Mock (gmock) are the de-facto standard C++ testing framework. Proper CMake integration ensures tests are discoverable, parallelizable, and work seamlessly with CI. Modern best practice uses `FetchContent` to pin a specific gtest version, avoiding system-installed mismatches.

### Integration Methods

| Method | Pros | Cons |
| --- | --- | --- |
| **FetchContent** (recommended) | Pinned version, no manual downloads | Slower first configure |
| Git submodule | Works offline after clone | Must manage submodule updates |
| `find_package` (system) | Fast configure | Version mismatches, not portable |
| Conan / vcpkg | Package manager benefits | Extra tooling dependency |

### Project Layout

```cpp

project/
├── CMakeLists.txt          # Top-level: project, FetchContent
├── src/
│   ├── CMakeLists.txt      # Library target
│   ├── calculator.h
│   └── calculator.cpp
└── tests/
    ├── CMakeLists.txt      # Test targets
    ├── calculator_test.cpp
    └── integration_test.cpp

```

---

## Self-Assessment

### Q1: Set up a complete CMake project with Google Test using FetchContent

**Answer:**

```cmake

# === CMakeLists.txt (top-level) ===
cmake_minimum_required(VERSION 3.14)
project(MyProject VERSION 1.0 LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# Production code
add_subdirectory(src)

# Tests — controlled by option so users can skip
option(BUILD_TESTING "Build the test suite" ON)
if(BUILD_TESTING)
    enable_testing()  # Must be in top-level CMakeLists.txt for ctest
    add_subdirectory(tests)
endif()

```

```cmake

# === src/CMakeLists.txt ===
add_library(calculator
    calculator.cpp
)
target_include_directories(calculator PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

```

```cpp

// === src/calculator.h ===
#pragma once

class Calculator {
public:
    double add(double a, double b) const;
    double divide(double a, double b) const;  // throws on div by zero
    double accumulate(double value);           // stateful
    double total() const;
private:
    double total_ = 0.0;
};

```

```cpp

// === src/calculator.cpp ===
#include "calculator.h"
#include <stdexcept>

double Calculator::add(double a, double b) const { return a + b; }

double Calculator::divide(double a, double b) const {
    if (b == 0.0) throw std::invalid_argument("Division by zero");
    return a / b;
}

double Calculator::accumulate(double value) {
    total_ += value;
    return total_;
}

double Calculator::total() const { return total_; }

```

```cmake

# === tests/CMakeLists.txt ===
include(FetchContent)

FetchContent_Declare(
    googletest
    GIT_REPOSITORY https://github.com/google/googletest.git
    GIT_TAG        v1.14.0  # Pin to specific release
)

# Prevent gtest from overriding our compiler settings on Windows
set(gtest_force_shared_crt ON CACHE BOOL "" FORCE)

FetchContent_MakeAvailable(googletest)

# Include GoogleTest module for gtest_discover_tests
include(GoogleTest)

# Unit test executable
add_executable(calculator_test
    calculator_test.cpp
)
target_link_libraries(calculator_test
    PRIVATE
        calculator       # Library under test
        GTest::gtest_main  # Provides main()
        GTest::gmock       # Google Mock
)

# Auto-discover tests for ctest
gtest_discover_tests(calculator_test)

```

```cpp

// === tests/calculator_test.cpp ===
#include <gtest/gtest.h>
#include "calculator.h"

TEST(CalculatorTest, AddPositiveNumbers) {
    Calculator calc;
    EXPECT_DOUBLE_EQ(calc.add(2.0, 3.0), 5.0);
}

TEST(CalculatorTest, DivideByZeroThrows) {
    Calculator calc;
    EXPECT_THROW(calc.divide(1.0, 0.0), std::invalid_argument);
}

TEST(CalculatorTest, AccumulateState) {
    Calculator calc;
    calc.accumulate(10.0);
    calc.accumulate(20.0);
    EXPECT_DOUBLE_EQ(calc.total(), 30.0);
}

```

```bash

# Build and run:
mkdir build && cd build
cmake .. -DBUILD_TESTING=ON
cmake --build .
ctest --output-on-failure -j$(nproc)

```

### Q2: What are the key configuration options and best practices

**Answer:**

| Practice | Why |
| --- | --- |
| Use `gtest_discover_tests` over `add_test` | Auto-discovers TEST macros; no manual registration |
| Pin `GIT_TAG` to a release | Reproducible builds |
| `gtest_force_shared_crt ON` on Windows | Avoids CRT mismatch linker errors |
| Link `GTest::gtest_main` | Don't write your own `main()` unless customizing |
| Use `EXPECT_*` not `ASSERT_*` by default | `EXPECT` continues test; `ASSERT` aborts the test case |
| Separate test executables by module | Parallel ctest execution; faster feedback |
| `BUILD_TESTING` option | Let consumers disable tests |

**Common pitfalls:**

```cmake

# BAD: Don't use add_test manually with gtest
add_test(NAME my_test COMMAND calculator_test)  # Discovers only 1 "test"

# GOOD: Auto-discover individual TEST() entries
gtest_discover_tests(calculator_test
    PROPERTIES TIMEOUT 30          # Per-test timeout
    DISCOVERY_TIMEOUT 10           # Discovery phase timeout
    WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
)

```

### Q3: Show advanced setup with multiple test suites, custom main, and CI integration

**Answer:**

```cmake

# === Advanced tests/CMakeLists.txt ===

# Shared test utilities library
add_library(test_utils STATIC test_helpers.cpp)
target_link_libraries(test_utils PUBLIC GTest::gtest GTest::gmock calculator)

# Unit tests
add_executable(unit_tests
    calculator_test.cpp
    parser_test.cpp
)
target_link_libraries(unit_tests PRIVATE test_utils GTest::gtest_main GTest::gmock)
gtest_discover_tests(unit_tests PROPERTIES LABELS "unit")

# Integration tests (may be slower)
add_executable(integration_tests
    database_integration_test.cpp
)
target_link_libraries(integration_tests PRIVATE test_utils GTest::gtest_main)
gtest_discover_tests(integration_tests PROPERTIES LABELS "integration")

# Custom main for tests that need global setup
add_executable(custom_main_tests
    custom_main_test.cpp
)
target_link_libraries(custom_main_tests PRIVATE test_utils GTest::gtest GTest::gmock)
gtest_discover_tests(custom_main_tests)

```

```cpp

// === custom_main_test.cpp ===
#include <gtest/gtest.h>
#include <gmock/gmock.h>

// Global test environment for one-time setup/teardown
class DatabaseEnvironment : public ::testing::Environment {
public:
    void SetUp() override {
        // Start embedded database, create schema, etc.
        std::cout << "[GLOBAL] Database initialized\n";
    }
    void TearDown() override {
        std::cout << "[GLOBAL] Database cleaned up\n";
    }
};

int main(int argc, char** argv) {
    ::testing::InitGoogleMock(&argc, argv);
    ::testing::AddGlobalTestEnvironment(new DatabaseEnvironment);
    return RUN_ALL_TESTS();
}

```

```bash

# CI: Run only unit tests, with JUnit XML output
ctest --test-dir build -L unit --output-junit results.xml --output-on-failure -j4

# Run with verbose output
ctest --test-dir build --verbose

# Run specific test by regex
ctest --test-dir build -R "Calculator.*Divide"

```

---

## Notes

- `GTest::gtest_main` provides a default `main()` — only write custom main for global setup
- `gtest_discover_tests` runs at build time (not configure time) — more robust than `gtest_add_tests`
- On Windows: always set `gtest_force_shared_crt ON` before `FetchContent_MakeAvailable`
- Use `PROPERTIES LABELS` with `ctest -L` for running subsets (unit vs integration)
- Consider `DISCOVERY_MODE PRE_TEST` for header-only tests where binary isn't available at build time
- Google Mock is included in googletest repo since 2019 — just link `GTest::gmock`
