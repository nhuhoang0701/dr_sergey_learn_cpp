# Use ctest with labels and parallel execution for large test suites

**Category:** Tooling & Debugging  
**Item:** #421  
**Reference:** <https://cmake.org/cmake/help/latest/manual/ctest.1.html>  

---

## Topic Overview

CTest is CMake's test runner. It supports **labels** for categorization and **parallel execution** for speed.

```bash

ctest --test-dir build -j8                 # run 8 tests in parallel
ctest --test-dir build -L fast             # run only "fast" labeled tests
ctest --test-dir build -L slow --timeout 60 # "slow" tests with timeout
ctest --test-dir build --output-on-failure  # show stdout/stderr on failure

```

---

## Self-Assessment

### Q1: Label tests and run subsets

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.20)
project(TestDemo LANGUAGES CXX)

set(CMAKE_CXX_STANDARD 20)
enable_testing()

# Fast unit tests
add_executable(test_math tests/test_math.cpp)
add_test(NAME MathBasic COMMAND test_math)
set_tests_properties(MathBasic PROPERTIES
    LABELS "fast;unit"
    TIMEOUT 5
)

add_executable(test_string tests/test_string.cpp)
add_test(NAME StringOps COMMAND test_string)
set_tests_properties(StringOps PROPERTIES
    LABELS "fast;unit"
    TIMEOUT 5
)

# Slow integration tests
add_executable(test_db tests/test_db.cpp)
add_test(NAME DatabaseIntegration COMMAND test_db)
set_tests_properties(DatabaseIntegration PROPERTIES
    LABELS "slow;integration"
    TIMEOUT 60
)

add_executable(test_network tests/test_network.cpp)
add_test(NAME NetworkE2E COMMAND test_network)
set_tests_properties(NetworkE2E PROPERTIES
    LABELS "slow;e2e"
    TIMEOUT 120
)

```

```bash

# Run only fast tests (for local development):
ctest --test-dir build -L fast
# Output:
#   Test #1: MathBasic ......... Passed  0.01 sec
#   Test #2: StringOps ......... Passed  0.02 sec
#   100% tests passed, 0 tests failed

# Run only slow tests (in CI nightly):
ctest --test-dir build -L slow

# Exclude slow tests:
ctest --test-dir build -LE slow

# Run tests matching a regex:
ctest --test-dir build -R "Math|String"

# List available tests and labels:
ctest --test-dir build -N

```

### Q2: Parallel execution and test dependencies

```cmake

# Parallel-safe tests (no shared state):
set_tests_properties(MathBasic StringOps PROPERTIES
    LABELS "fast;unit"
    PROCESSORS 1        # each test uses 1 CPU core
)

# Test that needs exclusive access to a resource:
set_tests_properties(DatabaseIntegration PROPERTIES
    LABELS "slow;integration"
    RESOURCE_LOCK "database"   # prevent parallel DB tests
)

# Test that depends on another test completing first:
set_tests_properties(NetworkE2E PROPERTIES
    DEPENDS DatabaseIntegration  # runs AFTER DatabaseIntegration
    LABELS "slow;e2e"
)

# Multi-core test:
set_tests_properties(HeavyComputation PROPERTIES
    PROCESSORS 4          # tells CTest this test uses 4 cores
    # CTest won't schedule more than (total_cores - 4) other tests simultaneously
)

```

```bash

# Run with 8 parallel jobs:
ctest --test-dir build -j8

# CTest parallel scheduling:
#   t=0:  MathBasic, StringOps, test3, test4, test5, test6, test7, test8
#   t=0.02: (MathBasic done) -> start test9
#   ...until all done
#
# Tests with DEPENDS wait for predecessors
# Tests with RESOURCE_LOCK serialize on that resource
# Tests with PROCESSORS > 1 reduce parallelism accordingly

# Verbose output:
ctest --test-dir build -j8 -V --output-on-failure

```

### Q3: Generate JUnit XML reports for CI

```bash

# Generate JUnit XML report:
ctest --test-dir build --output-junit test-results.xml --output-on-failure -j4

# The XML file is compatible with:
#   - GitHub Actions (upload-artifact)
#   - Jenkins (JUnit plugin)
#   - GitLab CI (artifacts:reports:junit)
#   - Azure DevOps (PublishTestResults)

```

CI integration example:

```yaml

# .github/workflows/test.yml
name: Tests
on: [push, pull_request]

jobs:
  test:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Configure

        run: cmake -B build -DCMAKE_BUILD_TYPE=Debug

      - name: Build

        run: cmake --build build -j$(nproc)

      - name: Test

        run: |
          cd build
          ctest --output-junit test-results.xml \
                --output-on-failure \
                -j$(nproc) \
                --timeout 120

      - name: Upload Test Results

        if: always()
        uses: actions/upload-artifact@v4
        with:
          name: test-results
          path: build/test-results.xml

```

---

## Notes

- `-j0` uses all available cores (auto-detect).
- `RESOURCE_LOCK` prevents tests sharing a resource from running in parallel.
- Use `FIXTURES_SETUP` / `FIXTURES_CLEANUP` for complex setup/teardown sequences.
- `--rerun-failed` re-runs only previously failed tests (fast iteration).
- `--repeat until-pass:3` retries flaky tests up to 3 times.
- Add `--no-tests=error` to fail the CI if no tests are found (catches config mistakes).
