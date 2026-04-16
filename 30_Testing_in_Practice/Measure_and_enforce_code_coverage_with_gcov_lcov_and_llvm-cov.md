# Measure and enforce code coverage with gcov, lcov, and llvm-cov

**Category:** Testing in Practice

---

## Topic Overview

Code coverage measures what percentage of your source code is executed during testing. It's a **necessary but not sufficient** quality metric — 100% coverage doesn't mean bug-free, but low coverage guarantees blind spots. The major tools for C++ are **gcov** (GCC), **llvm-cov** (Clang), and **lcov** (HTML report generator).

### Coverage Metrics

| Metric | What It Measures | Usefulness |
| --- | --- | --- |
| **Line coverage** | Lines executed / total lines | Basic — misses branch logic |
| **Branch coverage** | Branches taken / total branches | Better — catches untested if/else |
| **Function coverage** | Functions called / total functions | Finds dead code |
| **Region coverage** (llvm-cov) | Code regions executed | Most precise |
| **MC/DC** | Modified condition/decision | Required for safety-critical (DO-178C) |

### Tool Chain

```cpp

Source + Tests → Compile with coverage flags → Run tests
    │                                              │
    │           .gcno (compile-time notes)         │ .gcda (runtime data)
    │                                              │
    └────────► gcov / llvm-cov ─────────────────┘
                    │
              lcov / genhtml → HTML report

```

---

## Self-Assessment

### Q1: Set up coverage measurement with GCC/gcov and lcov

**Answer:**

```cmake

# === CMakeLists.txt: coverage build type ===
option(ENABLE_COVERAGE "Build with coverage instrumentation" OFF)

if(ENABLE_COVERAGE)
    if(CMAKE_CXX_COMPILER_ID MATCHES "GNU")
        add_compile_options(--coverage -fprofile-arcs -ftest-coverage)
        add_link_options(--coverage)
    elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
        add_compile_options(-fprofile-instr-generate -fcoverage-mapping)
        add_link_options(-fprofile-instr-generate)
    endif()
endif()

```

```bash

# === GCC + gcov + lcov workflow ===

# 1. Build with coverage
cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# 2. Run tests (generates .gcda files)
cd build && ctest --output-on-failure

# 3. Capture coverage data
lcov --capture --directory . --output-file coverage.info

# 4. Remove unwanted paths (system headers, test code)
lcov --remove coverage.info \
    '/usr/*' \
    '*/test/*' \
    '*/googletest/*' \
    --output-file coverage_filtered.info

# 5. Generate HTML report
genhtml coverage_filtered.info \
    --output-directory coverage_report \
    --title "MyProject Coverage" \
    --legend --highlight --demangle-cpp

# 6. Open report
# open coverage_report/index.html

# 7. Check coverage threshold (fails if below 80%)
lcov --summary coverage_filtered.info | grep -E 'lines|functions'
# Output: lines......: 85.3% (256 of 300 lines)

```

```bash

# === Per-file coverage with gcov ===
gcov -b -c src/calculator.cpp
# Produces calculator.cpp.gcov with line-by-line annotations:
#     1:   12: double Calculator::add(double a, double b) {
#     1:   13:     return a + b;
#     -:   14: }
#####:   15: double Calculator::unused_method() {
# ^^^^^ means line was never executed

```

### Q2: Set up coverage with Clang/llvm-cov

**Answer:**

```bash

# === Clang + llvm-cov workflow ===

# 1. Build with coverage
clang++ -fprofile-instr-generate -fcoverage-mapping \
    -o test_runner src/*.cpp tests/*.cpp -lgtest -lgtest_main

# 2. Run tests (generates default.profraw)
LLVM_PROFILE_FILE="test.profraw" ./test_runner

# 3. Merge profile data
llvm-profdata merge -sparse test.profraw -o test.profdata

# 4. Generate report
# Text summary:
llvm-cov report ./test_runner \
    -instr-profile=test.profdata \
    -ignore-filename-regex='test|googletest'

# Detailed source view:
llvm-cov show ./test_runner \
    -instr-profile=test.profdata \
    -format=html \
    -output-dir=coverage_report \
    -ignore-filename-regex='test|googletest' \
    -show-line-counts-or-regions \
    -show-branches=count

# 5. Export for CI tools:
llvm-cov export ./test_runner \
    -instr-profile=test.profdata \
    -format=lcov > coverage.lcov

```

```bash

# === Check specific file coverage ===
llvm-cov report ./test_runner \
    -instr-profile=test.profdata \
    src/calculator.cpp

# Output:
# Filename         Regions  Miss  Cover  Lines  Miss  Cover  Branches  Miss  Cover
# calculator.cpp        12     1  91.7%     45     3  93.3%        8     2  75.0%

```

### Q3: Enforce coverage thresholds in CI and integrate with CMake

**Answer:**

```cmake

# === CMake custom target for coverage ===
if(ENABLE_COVERAGE)
    find_program(LCOV lcov)
    find_program(GENHTML genhtml)

    add_custom_target(coverage
        # Reset counters
        COMMAND ${LCOV} --zerocounters --directory ${CMAKE_BINARY_DIR}
        # Run tests
        COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
        # Capture
        COMMAND ${LCOV} --capture
            --directory ${CMAKE_BINARY_DIR}
            --output-file ${CMAKE_BINARY_DIR}/coverage.info
        # Filter
        COMMAND ${LCOV} --remove ${CMAKE_BINARY_DIR}/coverage.info
            '/usr/*' '*/test/*' '*/build/*'
            --output-file ${CMAKE_BINARY_DIR}/coverage_filtered.info
        # Report
        COMMAND ${GENHTML} ${CMAKE_BINARY_DIR}/coverage_filtered.info
            --output-directory ${CMAKE_BINARY_DIR}/coverage_report
        WORKING_DIRECTORY ${CMAKE_BINARY_DIR}
        COMMENT "Generating code coverage report..."
    )
endif()

```

```yaml

# === GitHub Actions: enforce coverage threshold ===
name: Coverage
on: [push, pull_request]
jobs:
  coverage:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build with coverage

        run: |
          cmake -B build -DENABLE_COVERAGE=ON -DCMAKE_BUILD_TYPE=Debug
          cmake --build build -j$(nproc)

      - name: Run tests

        run: ctest --test-dir build --output-on-failure -j$(nproc)

      - name: Generate coverage

        run: |
          lcov --capture --directory build --output-file coverage.info
          lcov --remove coverage.info '/usr/*' '*/test/*' '*/googletest/*' \
               --output-file coverage_filtered.info

      - name: Enforce minimum coverage

        run: |
          COVERAGE=$(lcov --summary coverage_filtered.info 2>&1 | \
                     grep 'lines' | grep -oP '[0-9.]+(?=%)')
          echo "Line coverage: ${COVERAGE}%"
          if (( $(echo "$COVERAGE < 80" | bc -l) )); then
            echo "::error::Coverage ${COVERAGE}% is below threshold 80%"
            exit 1
          fi

      - name: Upload to Codecov

        uses: codecov/codecov-action@v4
        with:
          files: coverage_filtered.info
          fail_ci_if_error: true

```

---

## Notes

- **Coverage is a necessary but not sufficient metric** — it shows what code runs, not whether it's correct
- Aim for **80%+ line coverage** as a baseline; 90%+ for critical code
- **Branch coverage** is more valuable than line coverage — catches missed if/else paths
- Always filter out test code, third-party code, and generated code from metrics
- `--coverage` is shorthand for `-fprofile-arcs -ftest-coverage` (GCC)
- Use `LLVM_PROFILE_FILE` environment variable to control output location
- Coverage slows execution 2-5x — use dedicated coverage builds, not release builds
- `gcovr` is a Python alternative to lcov that produces Cobertura XML (for Jenkins/GitLab)
- Mutation testing (see separate topic) is a stronger quality metric than coverage
