# Apply the Single Responsibility Principle at the function level

**Category:** Best Practices & Idioms  
**Item:** #402  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Rf-single>  

---

## Topic Overview

The **Single Responsibility Principle (SRP)** at function level: *a function should do one thing, do it well, and do it only.* If you need the word "and" to describe what a function does, it likely violates SRP.

### The "And" Smell

```cpp

❌ parse_and_validate(input)       → Two responsibilities
❌ read_and_transform(file)        → Two responsibilities
❌ calculate_and_display(data)     → Two responsibilities

✅ parse(input)                    → One responsibility
✅ validate(parsed)                → One responsibility
✅ calculate(data)                 → One responsibility
✅ display(result)                 → One responsibility

```

---

## Self-Assessment

### Q1: Split a monolithic function into single-purpose functions

```cpp

#include <algorithm>
#include <iostream>
#include <sstream>
#include <stdexcept>
#include <string>
#include <vector>

// BEFORE: one function does EVERYTHING
std::string process_csv_bad(const std::string& raw) {
    // Parse
    std::vector<int> numbers;
    std::istringstream ss(raw);
    std::string token;
    while (std::getline(ss, token, ','))
        numbers.push_back(std::stoi(token));

    // Validate
    for (int n : numbers)
        if (n < 0) throw std::runtime_error("Negative value");

    // Transform
    std::sort(numbers.begin(), numbers.end());

    // Format output
    std::string result;
    for (size_t i = 0; i < numbers.size(); ++i) {
        if (i > 0) result += ", ";
        result += std::to_string(numbers[i]);
    }
    return result;
}

// AFTER: each function does ONE thing
std::vector<int> parse_csv(const std::string& raw) {
    std::vector<int> numbers;
    std::istringstream ss(raw);
    std::string token;
    while (std::getline(ss, token, ','))
        numbers.push_back(std::stoi(token));
    return numbers;
}

void validate_non_negative(const std::vector<int>& nums) {
    for (int n : nums)
        if (n < 0) throw std::runtime_error("Negative: " + std::to_string(n));
}

std::vector<int> sort_ascending(std::vector<int> nums) {
    std::sort(nums.begin(), nums.end());
    return nums;
}

std::string format_csv(const std::vector<int>& nums) {
    std::string result;
    for (size_t i = 0; i < nums.size(); ++i) {
        if (i > 0) result += ", ";
        result += std::to_string(nums[i]);
    }
    return result;
}

// Pipeline: compose single-purpose functions
std::string process_csv(const std::string& raw) {
    auto parsed   = parse_csv(raw);
    validate_non_negative(parsed);
    auto sorted   = sort_ascending(std::move(parsed));
    return format_csv(sorted);
}

int main() {
    std::cout << process_csv("30,10,50,20,40") << '\n';
}
// Expected output:
// 10, 20, 30, 40, 50

```

### Q2: Show that testing becomes simpler when each function has one reason to change

```cpp

#include <cassert>
#include <iostream>
#include <stdexcept>

// Using the single-purpose functions from Q1:
// Each can be tested in isolation

void test_parse_csv() {
    auto result = parse_csv("1,2,3");
    assert(result.size() == 3);
    assert(result[0] == 1 && result[1] == 2 && result[2] == 3);

    auto empty = parse_csv("");
    // handles edge case
}

void test_validate() {
    validate_non_negative({1, 2, 3});  // should not throw

    bool threw = false;
    try {
        validate_non_negative({1, -1, 3});
    } catch (const std::runtime_error&) {
        threw = true;
    }
    assert(threw);
}

void test_sort() {
    auto result = sort_ascending({3, 1, 2});
    assert(result[0] == 1 && result[1] == 2 && result[2] == 3);
}

void test_format() {
    assert(format_csv({1, 2, 3}) == "1, 2, 3");
    assert(format_csv({}) == "");
    assert(format_csv({42}) == "42");
}

int main() {
    test_parse_csv();
    test_validate();
    test_sort();
    test_format();
    std::cout << "All tests passed!\n";
}
// Expected output:
// All tests passed!

```

**Why this is better:**

- If parsing breaks, only `test_parse_csv` fails — immediate diagnosis.
- Validation logic changes? Only `test_validate` is affected.
- With the monolithic function, ANY failure requires debugging 30+ lines.

### Q3: Identify the "and" smell in function names

**Red flags in function names:**

```cpp

// BAD: "and" in the description = multiple responsibilities
void load_and_parse_config(const std::string& path);
void validate_and_save_user(User& u);
int  read_and_sum_values(std::istream& is);
void connect_and_authenticate(const std::string& host);

// GOOD: single responsibility per function
Config load_config(const std::string& path);
Config parse_config(const std::string& raw);
bool   validate_user(const User& u);
void   save_user(const User& u);
std::vector<int> read_values(std::istream& is);
int    sum(const std::vector<int>& values);
void   connect(const std::string& host);
void   authenticate(Connection& conn);

```

**Guidelines for function size:**

| Metric | Guideline |
| --- | --- |
| Lines of code | ≤20 (ideally ≤10) |
| Parameters | ≤4 |
| Nesting depth | ≤2 levels |
| Cyclomatic complexity | ≤5 |
| Can describe without "and" | Yes |

---

## Notes

- SRP doesn't mean "one line per function" — it means one **reason to change**.
- Extracting functions enables composition: `pipeline = parse | validate | transform | format`.
- C++ Core Guideline F.2: "A function should perform a single logical operation."
- Use concepts/templates to make extracted functions generic and reusable.
