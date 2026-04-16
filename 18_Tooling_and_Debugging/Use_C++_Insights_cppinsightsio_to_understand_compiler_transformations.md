# Use C++ Insights (cppinsights.io) to understand compiler transformations

**Category:** Tooling & Debugging  
**Item:** #800  
**Standard:** C++20  
**Reference:** <https://cppinsights.io>  

---

## Topic Overview

C++ Insights shows you what the **compiler actually generates** from your C++ code. It reveals the hidden transformations: range-for expansion, lambda closure classes, CTAD deduction guides, implicit conversions.

```cpp

Your code:                    Compiler sees:
  for (auto x : vec)    →     auto __begin = vec.begin();
                              auto __end = vec.end();
                              for (; __begin != __end; ++__begin) {
                                  auto x = *__begin;
                              }

```

---

## Self-Assessment

### Q1: Range-based for loop expansion

```cpp

// YOUR CODE (paste into cppinsights.io):
#include <vector>
#include <iostream>

int main() {
    std::vector<int> nums = {1, 2, 3, 4, 5};

    for (const auto& n : nums) {
        std::cout << n << ' ';
    }
}
// Expected output:
// 1 2 3 4 5

// C++ INSIGHTS REVEALS (approximately):
// {
//     std::vector<int>& __range = nums;
//     auto __begin = __range.begin();
//     auto __end = __range.end();
//     for (; __begin != __end; ++__begin) {
//         const int& n = *__begin;
//         std::cout.operator<<(n).operator<<(' ');
//     }
// }

```

Key insights:

- The range expression is evaluated **once** and bound to a reference.
- `begin()`/`end()` are called once, before the loop starts.
- The loop variable `n` is created by dereferencing the iterator each iteration.
- `auto&` becomes the exact iterator value type (`const int&` here).

### Q2: Lambda closure class synthesis

```cpp

#include <iostream>
#include <string>
#include <algorithm>
#include <vector>

int main() {
    int multiplier = 3;
    std::string prefix = "Result: ";

    // This lambda:
    auto formatter = [multiplier, &prefix](int x) -> std::string {
        return prefix + std::to_string(x * multiplier);
    };

    std::cout << formatter(7) << '\n';
}
// Expected output:
// Result: 21

// C++ INSIGHTS REVEALS the compiler-generated closure class:
//
// class __lambda_8_22 {
// public:
//     std::string operator()(int x) const {
//         return prefix + std::to_string(x * multiplier);
//     }
// private:
//     int multiplier;            // captured by VALUE (copy)
//     std::string& prefix;       // captured by REFERENCE
// public:
//     __lambda_8_22(int _multiplier, std::string& _prefix)
//         : multiplier(_multiplier), prefix(_prefix) {}
// };
//
// Key observations:
// - By-value capture = member variable (copy of original)
// - By-reference capture = reference member
// - operator() is const by default (no mutable)
// - mutable lambda -> operator() is non-const
// - Each lambda has a UNIQUE type (no two lambdas have the same type)

```

### Q3: CTAD (Class Template Argument Deduction) deduction guides

```cpp

#include <iostream>
#include <string>
#include <vector>

// Custom type with deduction guide:
template<typename T>
struct Wrapper {
    T value;
    Wrapper(T v) : value(v) {}
};

// Explicit deduction guide: string literals -> std::string
Wrapper(const char*) -> Wrapper<std::string>;

int main() {
    // CTAD in action:
    Wrapper w1(42);         // Wrapper<int>
    Wrapper w2(3.14);       // Wrapper<double>
    Wrapper w3("hello");    // Wrapper<std::string> (via deduction guide!)

    std::cout << w1.value << '\n';
    std::cout << w2.value << '\n';
    std::cout << w3.value << '\n';

    // STL CTAD:
    std::vector v = {1, 2, 3};           // vector<int>
    std::pair p = {42, "hello"};         // pair<int, const char*>

    std::cout << "vector size: " << v.size() << '\n';
    std::cout << "pair: (" << p.first << ", " << p.second << ")\n";
}
// Expected output:
// 42
// 3.14
// hello
// vector size: 3
// pair: (42, hello)

// C++ INSIGHTS REVEALS:
//   Wrapper<int> w1 = Wrapper<int>(42);
//   Wrapper<double> w2 = Wrapper<double>(3.14);
//   Wrapper<std::string> w3 = Wrapper<std::string>("hello");
//     ^-- deduction guide converted const char* -> std::string
//
//   std::vector<int> v = {1, 2, 3};
//     ^-- implicit deduction guide from initializer_list<int>
//
//   std::pair<int, const char*> p = {42, "hello"};
//     ^-- NOTE: const char*, NOT std::string!

```

---

## Notes

- C++ Insights is available at https://cppinsights.io (free, open-source).
- It shows: range-for expansion, lambdas, CTAD, auto deduction, coroutine frames.
- Use it to debug template errors — see what types the compiler actually deduced.
- It does NOT show optimizations (for that, use Godbolt/Compiler Explorer).
- C++ Insights uses Clang internally, so output reflects Clang's implementation.
