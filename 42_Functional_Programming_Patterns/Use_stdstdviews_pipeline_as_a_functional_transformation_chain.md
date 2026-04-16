# Use std::views pipeline as a functional transformation chain

**Category:** Functional Programming Patterns  
**Standard:** C++20/23  
**Reference:** <https://en.cppreference.com/w/cpp/ranges>  

---

## Topic Overview

`std::views` provides lazy, composable transformations over ranges — the C++ equivalent of functional map/filter/reduce pipelines.

### Basic Pipeline

```cpp

#include <ranges>
#include <vector>
#include <iostream>
#include <string>

int main() {
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // Functional pipeline: filter → transform → take
    auto result = data
        | std::views::filter([](int x) { return x % 2 == 0; })  // evens
        | std::views::transform([](int x) { return x * x; })    // square
        | std::views::take(3);                                    // first 3

    for (int x : result)
        std::cout << x << " ";  // 4 16 36
    std::cout << "\n";

    // LAZY: nothing computed until iteration
    // No intermediate vectors created
}

```

### Complex Pipeline with C++23

```cpp

#include <ranges>
#include <vector>
#include <string>
#include <iostream>

struct Employee {
    std::string name;
    std::string department;
    double salary;
};

void report(const std::vector<Employee>& employees) {
    // C++23: chunk_by, zip_transform, enumerate
    auto engineering_salaries = employees
        | std::views::filter([](const auto& e) {
            return e.department == "Engineering";
          })
        | std::views::transform(&Employee::salary)
        | std::views::filter([](double s) { return s > 80000; });

    // C++23 enumerate:
    for (auto [i, salary] : engineering_salaries | std::views::enumerate) {
        std::cout << i << ": $" << salary << "\n";
    }
}

```

### Composing Named Pipelines

```cpp

#include <ranges>

// Create reusable pipeline stages as variables
inline constexpr auto positive = std::views::filter([](auto x) { return x > 0; });
inline constexpr auto doubled = std::views::transform([](auto x) { return x * 2; });
inline constexpr auto first_five = std::views::take(5);

// Compose into a single pipeline
inline constexpr auto process = positive | doubled | first_five;

// Use:
// for (auto x : data | process) { ... }

```

---

## Self-Assessment

### Q1: Why are views lazy and why does it matter

Views don't compute anything until iterated. `data | filter(f) | transform(g) | take(3)` processes only 3 elements through the entire pipeline — it doesn't filter all elements first, then transform all, then take 3. This is O(1) extra memory and may process far fewer elements.

### Q2: What is the difference between views and actions

Views are lazy and non-owning — they don't modify the source range. Actions (range-v3) are eager and mutating — `sort | unique` would sort in-place and remove duplicates. C++ standard only has views; actions require range-v3 or manual algorithm calls.

### Q3: Show how to convert a view back to a container

```cpp

#include <ranges>
#include <vector>

auto view = std::views::iota(1, 10)
    | std::views::transform([](int x) { return x * x; });

// C++23: ranges::to
auto vec = view | std::ranges::to<std::vector>();

// C++20 workaround:
std::vector<int> vec20(view.begin(), view.end());

```

---

## Notes

- Views compose with `|` operator — read left to right, like Unix pipes.
- `std::views::iota`, `std::views::repeat`, `std::views::zip` are generators, not just filters.
- C++23 adds `ranges::to` for materializing views into containers.
- Views don't own data — ensure the source range outlives the view.
