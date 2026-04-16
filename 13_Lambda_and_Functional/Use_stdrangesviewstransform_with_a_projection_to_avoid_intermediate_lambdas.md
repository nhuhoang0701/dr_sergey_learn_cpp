# Use std::ranges::views::transform with a projection to avoid intermediate lambdas

**Category:** Lambda & Functional  
**Item:** #251  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/ranges/transform_view>  

---

## Topic Overview

In C++20, **member pointers** (both data members and member functions) can be used directly as callables with `views::transform`, `std::invoke`, and algorithm projections. This eliminates the need for trivial lambdas like `[](const auto& p) { return p.name; }`.

```cpp

struct Person { std::string name; int age; };
std::vector<Person> people = {{"Alice", 30}, {"Bob", 25}};

// Lambda (verbose):
for (auto& n : people | views::transform([](auto& p) { return p.name; }))
    std::cout << n;

// Member pointer (clean!):
for (auto& n : people | views::transform(&Person::name))
    std::cout << n;

```

---

## Self-Assessment

### Q1: Project a range of structs to a member using `views::transform(&T::member)`

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>
#include <numeric>

namespace rv = std::ranges::views;

struct Product {
    std::string name;
    double price;
    int quantity;
};

int main() {
    std::vector<Product> products = {
        {"Laptop", 999.99, 5},
        {"Mouse", 29.99, 50},
        {"Keyboard", 79.99, 30},
        {"Monitor", 449.99, 10}
    };

    // Project to names — NO lambda needed!
    std::cout << "Products:\n";
    for (const auto& name : products | rv::transform(&Product::name)) {
        std::cout << "  " << name << "\n";
    }

    // Project to prices:
    std::cout << "\nPrices:\n";
    for (double p : products | rv::transform(&Product::price)) {
        std::cout << "  $" << p << "\n";
    }

    // Use projection in algorithm:
    auto most_expensive = std::ranges::max(products, {}, &Product::price);
    std::cout << "\nMost expensive: " << most_expensive.name
              << " ($" << most_expensive.price << ")\n";

    // Sort by quantity (ascending):
    std::ranges::sort(products, {}, &Product::quantity);
    std::cout << "\nSorted by quantity:\n";
    for (const auto& p : products) {
        std::cout << "  " << p.name << ": " << p.quantity << "\n";
    }
}
// Expected output:
//   Products:
//     Laptop
//     Mouse
//     Keyboard
//     Monitor
//
//   Prices:
//     $999.99
//     $29.99
//     $79.99
//     $449.99
//
//   Most expensive: Laptop ($999.99)
//
//   Sorted by quantity:
//     Laptop: 5
//     Monitor: 10
//     Keyboard: 30
//     Mouse: 50

```

---

### Q2: Show that member function pointers work as projections too

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>

namespace rv = std::ranges::views;

struct Person {
    std::string first;
    std::string last;

    std::string full_name() const { return first + " " + last; }
    size_t name_length() const { return first.size() + last.size(); }
};

int main() {
    std::vector<Person> people = {
        {"Alice", "Smith"},
        {"Bob", "Johnson"},
        {"Charlie", "Brown"}
    };

    // Member FUNCTION pointer as projection — works with std::invoke!
    std::cout << "Full names:\n";
    for (const auto& name : people | rv::transform(&Person::full_name)) {
        std::cout << "  " << name << "\n";
    }

    // Member data pointer:
    std::cout << "\nFirst names:\n";
    for (const auto& f : people | rv::transform(&Person::first)) {
        std::cout << "  " << f << "\n";
    }

    // In algorithms with projection parameter:
    auto longest = std::ranges::max(people, {}, &Person::name_length);
    std::cout << "\nLongest name: " << longest.full_name() << "\n";

    // std::string::size works too:
    std::vector<std::string> words = {"short", "a", "medium-length", "hi"};
    std::ranges::sort(words, {}, &std::string::size);
    for (const auto& w : words) std::cout << w << " ";
    std::cout << "\n";
}
// Expected output:
//   Full names:
//     Alice Smith
//     Bob Johnson
//     Charlie Brown
//
//   First names:
//     Alice
//     Bob
//     Charlie
//
//   Longest name: Bob Johnson
//   a hi short medium-length

```

---

### Q3: Compose a filter and a projection in a single pipeline without intermediate containers

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <string>
#include <ranges>
#include <algorithm>

namespace rv = std::ranges::views;

struct Employee {
    std::string name;
    std::string dept;
    double salary;
};

int main() {
    std::vector<Employee> employees = {
        {"Alice", "Engineering", 95000},
        {"Bob", "Marketing", 72000},
        {"Charlie", "Engineering", 110000},
        {"Diana", "Marketing", 68000},
        {"Eve", "Engineering", 88000},
        {"Frank", "Sales", 75000}
    };

    // Pipeline: filter by dept → project to name → NO intermediate vector!
    std::cout << "Engineering team:\n";
    auto eng_names = employees
        | rv::filter([](const Employee& e) { return e.dept == "Engineering"; })
        | rv::transform(&Employee::name);

    for (const auto& name : eng_names) {
        std::cout << "  " << name << "\n";
    }

    // Filter + project to salary + compute sum
    double eng_total = 0;
    for (double s : employees
            | rv::filter([](const Employee& e) { return e.dept == "Engineering"; })
            | rv::transform(&Employee::salary)) {
        eng_total += s;
    }
    std::cout << "\nEngineering total salary: $" << eng_total << "\n";

    // Alternative: Use ranges algorithm with projection
    // No pipeline needed — projection IS the "transform"
    auto highest_paid = std::ranges::max(employees, {}, &Employee::salary);
    std::cout << "Highest paid: " << highest_paid.name
              << " ($" << highest_paid.salary << ")\n";

    // Complex pipeline: filter + transform + enumerate
    std::cout << "\nHigh earners (>$80k):\n";
    int idx = 1;
    for (const auto& name : employees
            | rv::filter([](const Employee& e) { return e.salary > 80000; })
            | rv::transform(&Employee::name)) {
        std::cout << "  " << idx++ << ". " << name << "\n";
    }
}
// Expected output:
//   Engineering team:
//     Alice
//     Charlie
//     Eve
//
//   Engineering total salary: $293000
//   Highest paid: Charlie ($110000)
//
//   High earners (>$80k):
//     1. Alice
//     2. Charlie
//     3. Eve

```

---

## Notes

- **Member pointer as projection** works because `std::invoke(&T::member, obj)` returns `obj.member` — both data members and member functions.
- **Algorithm projections** (second-to-last parameter in most ranges algorithms) accept the same callables as `views::transform`.
- **No copy overhead:** `views::transform(&T::member)` gives references to members, not copies.
- **Combine projections:** Chain transforms: `transform(&Employee::name) | transform(&string::size)`.
- **`std::identity`:** C++20's identity projection `{}` is the default — pass no projection to use elements as-is.
