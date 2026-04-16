# Write parameterized tests for table-driven test cases

**Category:** Testing & Verification  
**Item:** #685  
**Reference:** <https://google.github.io/googletest/advanced.html#value-parameterized-tests>  

---

## Topic Overview

**Parameterized tests** (a.k.a. table-driven tests) run the same test logic against multiple input/output pairs. Instead of writing N separate `TEST()` functions, you define one test body and a table of data.

### Comparison: Manual vs Parameterized

```cpp

Manual (6 tests, lots of duplication):          Parameterized (1 test, 6 rows):
TEST(Fizz, 3)  { EXPECT_EQ(fizz(3),  "Fizz"); }   TEST_P(FizzTest, Output) {
TEST(Fizz, 5)  { EXPECT_EQ(fizz(5),  "Buzz"); }       auto [in, out] = GetParam();
TEST(Fizz, 15) { EXPECT_EQ(fizz(15), "FizzBuzz"); }    EXPECT_EQ(fizz(in), out);
TEST(Fizz, 1)  { EXPECT_EQ(fizz(1),  "1"); }       }
TEST(Fizz, 0)  { EXPECT_EQ(fizz(0),  "0"); }       INSTANTIATE with 6 rows...
TEST(Fizz, -3) { EXPECT_EQ(fizz(-3), "Fizz"); }

```

### Framework Support

| Framework | Mechanism | Syntax |
| --- | --- | --- |
| Google Test | `TEST_P` + `INSTANTIATE_TEST_SUITE_P` | Class-based |
| Catch2 | `GENERATE()` | Inline generator |
| doctest | `TEST_CASE_TEMPLATE` / SUBCASE loop | Template or loop |
| Boost.Test | `BOOST_DATA_TEST_CASE` | Data-driven |

---

## Self-Assessment

### Q1: Use INSTANTIATE_TEST_SUITE_P in Google Test to run a test over a vector of input/output pairs

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <string>
#include <tuple>

// ═══════════ Function under test ═══════════
std::string classify_bmi(double bmi) {
    if (bmi < 18.5) return "Underweight";
    if (bmi < 25.0) return "Normal";
    if (bmi < 30.0) return "Overweight";
    return "Obese";
}

// ═══════════ Step 1: Define the test fixture ═══════════
class BmiTest : public ::testing::TestWithParam<std::tuple<double, std::string>> {};

// ═══════════ Step 2: Write the parameterized test ═══════════
TEST_P(BmiTest, ClassifiesCorrectly) {
    auto [bmi, expected] = GetParam();
    EXPECT_EQ(classify_bmi(bmi), expected);
}

// ═══════════ Step 3: Instantiate with the data table ═══════════
INSTANTIATE_TEST_SUITE_P(
    BmiClassification,         // Prefix for test names
    BmiTest,                   // Test fixture class
    ::testing::Values(
        std::make_tuple(15.0,  "Underweight"),
        std::make_tuple(18.4,  "Underweight"),
        std::make_tuple(18.5,  "Normal"),       // Boundary
        std::make_tuple(22.0,  "Normal"),
        std::make_tuple(24.9,  "Normal"),
        std::make_tuple(25.0,  "Overweight"),   // Boundary
        std::make_tuple(29.9,  "Overweight"),
        std::make_tuple(30.0,  "Obese"),        // Boundary
        std::make_tuple(40.0,  "Obese")
    ),
    // Optional: custom name generator for readable output
    [](const auto& info) {
        auto bmi = std::get<0>(info.param);
        return "BMI_" + std::to_string(static_cast<int>(bmi * 10));
    }
);

// Output:
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_150
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_184
// BmiClassification/BmiTest.ClassifiesCorrectly/BMI_185
// ...

// ═══════════ Multiple parameters ═══════════
struct ConversionCase {
    double celsius;
    double expected_fahrenheit;
    std::string name;
};

class TempTest : public ::testing::TestWithParam<ConversionCase> {};

TEST_P(TempTest, CelsiusToFahrenheit) {
    auto c = GetParam();
    double result = c.celsius * 9.0 / 5.0 + 32.0;
    EXPECT_NEAR(result, c.expected_fahrenheit, 0.01);
}

INSTANTIATE_TEST_SUITE_P(
    TemperatureConversions,
    TempTest,
    ::testing::Values(
        ConversionCase{0.0,   32.0,  "Freezing"},
        ConversionCase{100.0, 212.0, "Boiling"},
        ConversionCase{-40.0, -40.0, "Equal"},
        ConversionCase{37.0,  98.6,  "Body"}
    ),
    [](const auto& info) { return info.param.name; }
);

```

### Q2: Use GENERATE in Catch2 to parameterize a TEST_CASE over multiple values

**Answer:**

```cpp

#include <catch2/catch_test_macros.hpp>
#include <catch2/generators/catch_generators.hpp>
#include <catch2/generators/catch_generators_adapters.hpp>
#include <string>
#include <algorithm>

// ═══════════ Function under test ═══════════
bool is_palindrome(const std::string& s) {
    std::string lower = s;
    std::transform(lower.begin(), lower.end(), lower.begin(), ::tolower);
    std::string rev = lower;
    std::reverse(rev.begin(), rev.end());
    return lower == rev;
}

// ═══════════ GENERATE: inline parameterization ═══════════
TEST_CASE("Palindrome detection") {
    // GENERATE creates a parameter — test runs once per value
    auto [input, expected] = GENERATE(table<std::string, bool>({
        {"racecar",     true},
        {"Madam",       true},
        {"hello",       false},
        {"a",           true},
        {"",            true},
        {"ab",          false},
        {"AaBbAa",      false},  // case-sensitive? our impl is case-insensitive
    }));

    CAPTURE(input);  // Show input in failure message
    REQUIRE(is_palindrome(input) == expected);
}

// ═══════════ GENERATE with ranges and combinations ═══════════
TEST_CASE("Multiplication table boundaries") {
    auto a = GENERATE(range(1, 11));   // 1 through 10
    auto b = GENERATE(range(1, 11));   // 1 through 10 (cartesian product!)

    REQUIRE(a * b == b * a);           // Commutativity
    REQUIRE(a * b > 0);                // Both positive → product positive
}
// Runs 100 times (10 × 10)

// ═══════════ GENERATE with filter ═══════════
TEST_CASE("Division by non-zero") {
    auto divisor = GENERATE(filter(
        [](int x) { return x != 0; },  // Skip zero
        range(-10, 11)
    ));

    REQUIRE(100 / divisor != 0);  // No division by zero
}

// ═══════════ GENERATE with map ═══════════
TEST_CASE("String length categories") {
    auto input = GENERATE(as<std::string>{},
        "", "a", "ab", "abc", "abcde", "abcdefghij"
    );

    auto len = input.length();
    if (len == 0) {
        REQUIRE(input.empty());
    } else {
        REQUIRE(input[0] != '\0');
    }
}

```

### Q3: Show how parameterized tests reduce boilerplate vs writing one test per case

**Answer:**

```cpp

// ═══════════ BEFORE: One test per case (75 lines) ═══════════
TEST(HttpStatus, Ok)           { EXPECT_EQ(status_text(200), "OK"); }
TEST(HttpStatus, Created)      { EXPECT_EQ(status_text(201), "Created"); }
TEST(HttpStatus, BadRequest)   { EXPECT_EQ(status_text(400), "Bad Request"); }
TEST(HttpStatus, Unauthorized) { EXPECT_EQ(status_text(401), "Unauthorized"); }
TEST(HttpStatus, Forbidden)    { EXPECT_EQ(status_text(403), "Forbidden"); }
TEST(HttpStatus, NotFound)     { EXPECT_EQ(status_text(404), "Not Found"); }
TEST(HttpStatus, ServerError)  { EXPECT_EQ(status_text(500), "Internal Server Error"); }
TEST(HttpStatus, BadGateway)   { EXPECT_EQ(status_text(502), "Bad Gateway"); }
TEST(HttpStatus, Unavailable)  { EXPECT_EQ(status_text(503), "Service Unavailable"); }
// Problems:
//   - Adding a new status = copy-paste a line (easy to introduce typos)
//   - Logic is duplicated 9 times
//   - Can't easily loop over all cases for exhaustive check

// ═══════════ AFTER: Parameterized (25 lines) ═══════════
struct StatusCase {
    int code;
    std::string text;
};

class HttpStatusTest : public ::testing::TestWithParam<StatusCase> {};

TEST_P(HttpStatusTest, ReturnsCorrectText) {
    auto c = GetParam();
    EXPECT_EQ(status_text(c.code), c.text);
}

INSTANTIATE_TEST_SUITE_P(AllStatuses, HttpStatusTest, ::testing::Values(
    StatusCase{200, "OK"},
    StatusCase{201, "Created"},
    StatusCase{400, "Bad Request"},
    StatusCase{401, "Unauthorized"},
    StatusCase{403, "Forbidden"},
    StatusCase{404, "Not Found"},
    StatusCase{500, "Internal Server Error"},
    StatusCase{502, "Bad Gateway"},
    StatusCase{503, "Service Unavailable"}
));

// Benefits:
// ✓ Add new case = add one line to the table
// ✓ Test logic written ONCE
// ✓ Each row runs as a separate test (individual pass/fail)
// ✓ Easy to generate from external data

// ═══════════ Even better: combine with typed tests ═══════════
// Google Test TYPED_TEST for testing same interface across implementations:

template <typename T>
class ContainerTest : public ::testing::Test {};

using ContainerTypes = ::testing::Types<
    std::vector<int>, std::deque<int>, std::list<int>
>;
TYPED_TEST_SUITE(ContainerTest, ContainerTypes);

TYPED_TEST(ContainerTest, PushBackIncreasesSize) {
    TypeParam container;
    container.push_back(42);
    EXPECT_EQ(container.size(), 1u);
    container.push_back(99);
    EXPECT_EQ(container.size(), 2u);
}
// Runs for vector, deque, AND list — zero duplication

```

---

## Notes

- Google Test: `TEST_P` for value params, `TYPED_TEST` for type params — can combine both
- Catch2 `GENERATE()` is lazier — no fixture class needed, more concise
- Custom name generators prevent cryptic test names like `/0`, `/1`, `/2`
- Table-driven tests excel for pure functions; for stateful tests, use fixtures
- Parameterized tests make it trivial to add edge cases — just add a row to the table
- `::testing::Combine(Values(...), Values(...))` creates cartesian products in Google Test
