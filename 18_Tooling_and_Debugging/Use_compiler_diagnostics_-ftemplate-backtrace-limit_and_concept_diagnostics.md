# Use compiler diagnostics: -ftemplate-backtrace-limit and concept diagnostics

**Category:** Tooling & Debugging  
**Item:** #799  
**Standard:** C++20  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/C_002b_002b-Dialect-Options.html>  

---

## Topic Overview

C++ template errors produce notoriously long error messages. Key flags to control diagnostics:

| Flag | Compiler | Effect |
| --- | --- | --- |
| `-ftemplate-backtrace-limit=N` | GCC/Clang | Show N levels of template instantiation (0 = all) |
| `-fconcepts-diagnostics-depth=N` | GCC | Show N levels of concept constraint failures |
| `-fdiagnostics-show-template-tree` | Clang | Tree view of template type differences |

---

## Self-Assessment

### Q1: `-ftemplate-backtrace-limit=0` for full instantiation chain

```cpp

#include <vector>
#include <algorithm>
#include <list>

struct Widget {
    int x;
    // No operator< defined!
};

int main() {
    std::vector<Widget> v = {{3}, {1}, {2}};
    std::sort(v.begin(), v.end());  // ERROR deep inside <algorithm>
}
// Default compilation (limit=10):
//   error: no match for 'operator<' in ...
//   note: ... 5 more levels truncated ...
//
// With -ftemplate-backtrace-limit=0:
//   Shows the FULL chain:
//   main() -> std::sort() -> __sort() -> __move_median_to_first()
//          -> __comp() -> operator<(Widget, Widget) <- NOT FOUND
//
// The full chain helps you understand WHERE the error originates.

```

```bash

# Compile with unlimited backtrace:
g++ -std=c++20 -ftemplate-backtrace-limit=0 sort_error.cpp

# Clang equivalent:
clang++ -std=c++20 -ftemplate-backtrace-limit=0 sort_error.cpp

# CMake:
target_compile_options(myapp PRIVATE -ftemplate-backtrace-limit=0)

```

### Q2: SFINAE error vs concept constraint violation

```cpp

#include <type_traits>
#include <concepts>
#include <iostream>
#include <string>

// ========= SFINAE version =========
template<typename T,
         typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T add_sfinae(T a, T b) { return a + b; }

// Error when called with std::string:
//   add_sfinae(std::string("a"), std::string("b"));
// GCC SFINAE error:
//   error: no matching function for call to 'add_sfinae(std::string, std::string)'
//   note: candidate: template<class T, class> T add_sfinae(T, T)
//   note: template argument deduction/substitution failed:
//   note: ... std::enable_if_t<false> is not a type ...
//   ^^^^^^^^^^ CRYPTIC! What constraint failed? Why?

// ========= Concept version =========
template<std::integral T>    // clear constraint
T add_concept(T a, T b) { return a + b; }

// Error when called with std::string:
//   add_concept(std::string("a"), std::string("b"));
// GCC concept error:
//   error: no matching function for call to 'add_concept(std::string, std::string)'
//   note: the required expression 'std::integral<T>' is not satisfied
//   note: because 'std::is_integral_v<std::string>' evaluated to 'false'
//   ^^^^^^^^^^ CLEAR! Tells you exactly which constraint and why.

int main() {
    std::cout << add_sfinae(1, 2) << '\n';    // OK
    std::cout << add_concept(3, 4) << '\n';   // OK
}
// Expected output:
// 3
// 7

```

### Q3: `-fconcepts-diagnostics-depth` for nested constraints

```cpp

#include <concepts>
#include <ranges>
#include <vector>
#include <iostream>

// Deeply nested concept:
template<typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::same_as<std::ostream&>;
};

template<typename R>
concept PrintableRange = std::ranges::range<R> &&
    Printable<std::ranges::range_value_t<R>>;

template<PrintableRange R>
void print_all(const R& range) {
    for (const auto& x : range)
        std::cout << x << ' ';
    std::cout << '\n';
}

struct NoPrint { int x; };  // no operator<<

int main() {
    std::vector<int> v = {1, 2, 3};
    print_all(v);  // OK

    // std::vector<NoPrint> np = {{1}, {2}};
    // print_all(np);  // ERROR:
    // Default depth=1: "PrintableRange<vector<NoPrint>> not satisfied"
    // With -fconcepts-diagnostics-depth=3:
    //   because 'Printable<NoPrint>' is not satisfied
    //     because the expression 'os << t' is not valid
    //       because no operator<< matches 'ostream& << NoPrint'
}
// Expected output:
// 1 2 3

```

```bash

# Increase concept diagnostic depth:
g++ -std=c++20 -fconcepts-diagnostics-depth=3 concepts.cpp

# Clang has good concept diagnostics by default
# For template diff tree (Clang only):
clang++ -std=c++20 -fdiagnostics-show-template-tree concepts.cpp

```

---

## Notes

- `-ftemplate-backtrace-limit=0` is useful for debugging, but produces very long output.
- Concepts (C++20) provide dramatically better error messages than SFINAE.
- GCC's `-fconcepts-diagnostics-depth` defaults to 1; increase for nested constraints.
- Clang's `-fdiagnostics-show-template-tree` shows type diffs as a tree.
- Consider adding these flags only for development, not for release builds.
