# Write point-free style function composition using higher-order functions

**Category:** Lambda & Functional  
**Item:** #390  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional>  

---

## Topic Overview

**Point-free style** (also called tacit programming) defines functions without explicitly mentioning their arguments. Instead of `[](int x) { return f(g(x)); }`, you write `compose(f, g)`. This is natural in Haskell/F# but requires explicit combinators in C++.

```cpp

// Pointed style (mentions arguments):
auto process = [](int x) { return to_string(negate(double_it(x))); };

// Point-free style (no argument mentioned):
auto process = compose(to_string, compose(negate, double_it));
// or with pipeline:
auto process = pipeline(double_it, negate, to_string);

```

---

## Self-Assessment

### Q1: Implement `compose(f, g)` that returns `[](auto x) { return f(g(x)); }`

**Solution:**

```cpp

#include <iostream>
#include <string>
#include <cmath>

// compose(f, g)(x) = f(g(x))  — right-to-left (mathematical convention)
template <typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto&&... args) -> decltype(auto) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

// Variadic compose: compose_all(f, g, h)(x) = f(g(h(x)))
template <typename F>
auto compose_all(F f) { return f; }

template <typename F, typename... Fs>
auto compose_all(F f, Fs... rest) {
    return compose(f, compose_all(rest...));
}

int main() {
    auto square    = [](double x) { return x * x; };
    auto add_one   = [](double x) { return x + 1; };
    auto to_string = [](double x) { return std::to_string(x); };
    auto prefix    = [](std::string s) { return "Result: " + s; };

    // Two-function composition:
    auto square_then_add = compose(add_one, square);  // add_one(square(x))
    std::cout << square_then_add(5) << "\n";          // 5^2 + 1 = 26

    // Multi-function composition (right-to-left):
    auto pipeline = compose_all(prefix, to_string, add_one, square);
    // pipeline(3) = prefix(to_string(add_one(square(3))))
    //             = prefix(to_string(add_one(9)))
    //             = prefix(to_string(10))
    //             = prefix("10.000000")
    //             = "Result: 10.000000"
    std::cout << pipeline(3) << "\n";

    // Point-free: no argument name appears in the definition!
    auto abs_squared = compose(square, [](double x) { return std::abs(x); });
    std::cout << abs_squared(-7) << "\n";  // 49
}
// Expected output:
//   26
//   Result: 10.000000
//   49

```

---

### Q2: Use compose to build a string transformation pipeline without intermediate variables

**Solution:**

```cpp

#include <iostream>
#include <string>
#include <algorithm>
#include <cctype>

template <typename F, typename G>
auto compose(F f, G g) {
    return [f, g](auto&&... args) -> decltype(auto) {
        return f(g(std::forward<decltype(args)>(args)...));
    };
}

template <typename F>
auto pipeline(F f) { return f; }

template <typename F, typename... Fs>
auto pipeline(F f, Fs... rest) {
    return [f, ...rest_fns = rest](auto&&... args) -> decltype(auto) {
        return pipeline(rest...)(f(std::forward<decltype(args)>(args)...));
    };
}

int main() {
    // Point-free string transformations:
    auto trim_leading = [](std::string s) {
        s.erase(s.begin(), std::find_if(s.begin(), s.end(),
            [](char c) { return !std::isspace(c); }));
        return s;
    };

    auto to_upper = [](std::string s) {
        std::transform(s.begin(), s.end(), s.begin(), ::toupper);
        return s;
    };

    auto add_brackets = [](std::string s) {
        return "[" + s + "]";
    };

    auto truncate_10 = [](std::string s) {
        if (s.size() > 10) s.resize(10);
        return s;
    };

    // Build pipeline without mentioning arguments:
    auto format_tag = pipeline(trim_leading, to_upper, truncate_10, add_brackets);

    // Use:
    std::cout << format_tag("  hello world") << "\n";
    std::cout << format_tag("  short") << "\n";
    std::cout << format_tag("   this is a very long tag name") << "\n";
}
// Expected output:
//   [HELLO WORL]
//   [SHORT]
//   [THIS IS A ]

```

---

### Q3: Explain why C++ lacks built-in composition and what `ranges::views` fills

**Solution:**

```cpp

#include <iostream>
#include <vector>
#include <ranges>
#include <algorithm>
#include <string>

namespace rv = std::ranges::views;

int main() {
    // C++ has NO built-in function composition operator.
    // Haskell:  f . g           -- composition
    // F#:       x |> f |> g     -- pipeline
    // C++:      ???             -- must build manually

    // WHY C++ lacks it:
    // 1. C++ functions aren't first-class values
    //    - Function templates can't be passed as arguments
    //    - Overload sets aren't objects
    //    - ADL complicates lookup
    // 2. No unified callable concept until C++20's std::invocable
    // 3. Type deduction challenges with overloaded/templated functions

    // WHAT ranges::views provides: PIPELINE composition via operator|
    std::vector<int> data = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};

    // This IS point-free style for range operations!
    auto pipeline = rv::filter([](int x) { return x % 2 == 0; })
                  | rv::transform([](int x) { return x * x; })
                  | rv::take(3);

    std::cout << "Pipeline result: ";
    for (int x : data | pipeline) {
        std::cout << x << " ";
    }
    std::cout << "\n";

    // ranges::views::transform is essentially "map" composition:
    auto names = std::vector<std::string>{"Alice", "Bob", "Charlie"};
    auto lengths = rv::transform(&std::string::size);  // point-free!

    std::cout << "Lengths: ";
    for (auto len : names | lengths) {
        std::cout << len << " ";
    }
    std::cout << "\n";

    // Summary table:
    std::cout << "\n";
    std::cout << "Language    Composition   Pipeline\n";
    std::cout << "---------- ------------- -----------\n";
    std::cout << "Haskell     f . g         x & f & g\n";
    std::cout << "F#          f >> g        x |> f |> g\n";
    std::cout << "C++ ranges  N/A           r | f | g\n";
    std::cout << "C++ manual  compose(f,g)  pipe(f,g)(x)\n";
}
// Expected output:
//   Pipeline result: 4 16 36
//   Lengths: 5 3 7
//
//   Language    Composition   Pipeline
//   ---------- ------------- -----------
//   Haskell     f . g         x & f & g
//   F#          f >> g        x |> f |> g
//   C++ ranges  N/A           r | f | g
//   C++ manual  compose(f,g)  pipe(f,g)(x)

```

---

## Notes

- **`operator|` for ranges** is the closest C++ gets to built-in pipeline composition.
- **Boost.Hof** and **range-v3** provide `compose`, `partial`, and other combinators.
- **C++23 `std::bind_back`** helps with partial application, moving toward more point-free style.
- **Limitation:** C++ compose requires concrete callable types — you can't compose overload sets or function templates without wrapping them in lambdas first.
- **Performance:** Composition via lambdas is zero-overhead — the compiler inlines everything.
