# Understand How to Detect Whether a Type Is a Specialization of a Template

**Category:** Templates & Generic Programming  
**Item:** #452  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/partial_specialization>  

---

## Topic Overview

### The Problem

Given a type `T`, you want to check at compile time: "Is `T` some specialization of `std::vector`?" — i.e., `std::vector<int>`, `std::vector<string>`, etc.

### The `is_specialization_of` Trait

```cpp

// Primary: false by default
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

// Partial specialization: true when T matches Template<Args...>
template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

```

| Expression | Result |
| --- | :---: |
| `is_specialization_of<std::vector<int>, std::vector>` | `true` |
| `is_specialization_of<std::optional<int>, std::optional>` | `true` |
| `is_specialization_of<int, std::vector>` | `false` |
| `is_specialization_of<std::vector<int>, std::optional>` | `false` |

### Limitation: Non-Type Template Parameters

This technique only works with templates that take **type** parameters. It breaks for `std::array<T,N>` (has a non-type `N`) and `std::integer_sequence<T, Ns...>`.

---

## Self-Assessment

### Q1: Write `is_specialization_of<T, std::vector>` that is true only when `T` is a `std::vector`

```cpp

#include <iostream>
#include <type_traits>
#include <vector>
#include <optional>
#include <map>
#include <string>
#include <variant>

// Primary template: false
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

// Partial specialization: true when T = Template<Args...>
template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

// Helper variable template
template <typename T, template <typename...> class Template>
constexpr bool is_specialization_of_v = is_specialization_of<T, Template>::value;

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== is_specialization_of ===\n\n";

    // vector tests
    std::cout << "vector<int>       is vector:   "
              << is_specialization_of_v<std::vector<int>, std::vector> << "\n";        // true
    std::cout << "vector<string>    is vector:   "
              << is_specialization_of_v<std::vector<std::string>, std::vector> << "\n"; // true
    std::cout << "vector<int>       is optional: "
              << is_specialization_of_v<std::vector<int>, std::optional> << "\n";       // false

    // optional tests
    std::cout << "optional<int>     is optional: "
              << is_specialization_of_v<std::optional<int>, std::optional> << "\n";     // true

    // map test (has multiple type params — still works!)
    std::cout << "map<string,int>   is map:      "
              << is_specialization_of_v<std::map<std::string,int>, std::map> << "\n";   // true

    // non-template types
    std::cout << "int               is vector:   "
              << is_specialization_of_v<int, std::vector> << "\n";                      // false
    std::cout << "double            is optional:  "
              << is_specialization_of_v<double, std::optional> << "\n";                 // false

    // variant
    std::cout << "variant<int,str>  is variant:  "
              << is_specialization_of_v<std::variant<int,std::string>, std::variant> << "\n"; // true

    return 0;
}

```

**Expected output:**

```text

=== is_specialization_of ===

vector<int>       is vector:   true
vector<string>    is vector:   true
vector<int>       is optional: false
optional<int>     is optional: true
map<string,int>   is map:      true
int               is vector:   false
double            is optional:  false
variant<int,str>  is variant:  true

```

### Q2: Show the limitation: this approach breaks for non-type template parameter specializations

```cpp

#include <iostream>
#include <type_traits>
#include <array>

// Our is_specialization_of (type-only version)
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

template <typename T, template <typename...> class Template>
constexpr bool is_specialization_of_v = is_specialization_of<T, Template>::value;

// std::array<T, N> takes a TYPE (T) and a NON-TYPE (N = size_t)
// template<typename...> class Template can only match TYPE parameters!

// This will NOT compile:
// is_specialization_of_v<std::array<int, 5>, std::array>
// Error: std::array expects <typename, size_t>, not <typename...>

// Workaround for specific templates with non-type params:
template <typename T>
struct is_std_array : std::false_type {};

template <typename T, std::size_t N>
struct is_std_array<std::array<T, N>> : std::true_type {};

template <typename T>
constexpr bool is_std_array_v = is_std_array<T>::value;

// Similarly for std::integer_sequence:
template <typename T>
struct is_integer_sequence : std::false_type {};

template <typename T, T... Ints>
struct is_integer_sequence<std::integer_sequence<T, Ints...>> : std::true_type {};

int main() {
    std::cout << std::boolalpha;

    std::cout << "=== Limitation: non-type template parameters ===\n\n";

    // This WORKS (type-only parameters):
    // std::cout << is_specialization_of_v<std::vector<int>, std::vector>;  // true

    // This FAILS to compile:
    // std::cout << is_specialization_of_v<std::array<int,5>, std::array>;
    // Error: template template argument has different template params

    std::cout << "Broken: is_specialization_of<array<int,5>, array>  → compile error\n";
    std::cout << "Reason: std::array<T,N> has non-type param N (size_t)\n";
    std::cout << "        template<typename...> can only match type params\n\n";

    // Workaround: dedicated trait
    std::cout << "Workaround: is_std_array<T>\n";
    std::cout << "array<int,5>:   " << is_std_array_v<std::array<int,5>> << "\n";    // true
    std::cout << "array<float,3>: " << is_std_array_v<std::array<float,3>> << "\n";  // true
    std::cout << "int:            " << is_std_array_v<int> << "\n";                   // false

    std::cout << "\nOther templates with non-type params:\n";
    std::cout << "  std::array<T, N>              (size_t N)\n";
    std::cout << "  std::integer_sequence<T, Is>  (T... Is)\n";
    std::cout << "  std::bitset<N>                (size_t N)\n";
    std::cout << "  std::span<T, Extent>          (size_t Extent)\n";
    std::cout << "Each needs a dedicated trait.\n";

    return 0;
}

```

### Q3: Use a concept to constrain a function to accept only `std::optional` specializations

```cpp

#include <iostream>
#include <concepts>
#include <optional>
#include <string>
#include <vector>

// is_specialization_of trait
template <typename T, template <typename...> class Template>
struct is_specialization_of : std::false_type {};

template <template <typename...> class Template, typename... Args>
struct is_specialization_of<Template<Args...>, Template> : std::true_type {};

// Concept: T must be a std::optional<X> for some X
template <typename T>
concept IsOptional = is_specialization_of<std::remove_cvref_t<T>, std::optional>::value;

// Similarly for vector
template <typename T>
concept IsVector = is_specialization_of<std::remove_cvref_t<T>, std::vector>::value;

// Function constrained to only accept std::optional<T>
template <IsOptional Opt>
void unwrap_or_print(const Opt& opt, const typename Opt::value_type& fallback) {
    if (opt.has_value()) {
        std::cout << "Has value: " << *opt << "\n";
    } else {
        std::cout << "No value, using fallback: " << fallback << "\n";
    }
}

// Function constrained to only accept std::vector<T>
template <IsVector V>
void print_first(const V& v) {
    if (!v.empty()) {
        std::cout << "First element: " << v.front() << "\n";
    }
}

int main() {
    std::cout << "=== Concept-constrained to std::optional ===\n\n";

    std::optional<int> a = 42;
    std::optional<int> b = std::nullopt;
    std::optional<std::string> c = "hello";

    unwrap_or_print(a, 0);           // Has value: 42
    unwrap_or_print(b, -1);          // No value, using fallback: -1
    unwrap_or_print(c, std::string("default")); // Has value: hello

    // These would NOT compile — int is not an optional:
    // unwrap_or_print(42, 0);       // ERROR: constraint not satisfied
    // unwrap_or_print("hello", ""); // ERROR: constraint not satisfied

    std::cout << "\n=== Concept-constrained to std::vector ===\n\n";

    std::vector<int> v{10, 20, 30};
    print_first(v);  // First element: 10

    // print_first(42);  // ERROR: constraint not satisfied

    return 0;
}

```

**Expected output:**

```text

=== Concept-constrained to std::optional ===

Has value: 42
No value, using fallback: -1
Has value: hello

=== Concept-constrained to std::vector ===

First element: 10

```

---

## Notes

- `is_specialization_of<T, Template>` uses partial specialization to match `Template<Args...>`.
- Works for templates with **only type parameters** (`vector`, `optional`, `map`, `variant`).
- **Breaks** for templates with non-type parameters (`array<T,N>`, `bitset<N>`, `span<T,E>`) — use dedicated traits.
- Wrap in a concept for clean constrained overloads: `concept IsOptional = is_specialization_of<T, std::optional>::value;`
- Always use `std::remove_cvref_t<T>` in the concept to handle const/ref-qualified types.
