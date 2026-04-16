# Write `constexpr` Functions That Work Both at Compile Time and Runtime

**Category:** Compile-Time Programming  
**Item:** #53  
**Standard:** C++11 (expanded in C++14, C++17, C++20)  
**Reference:** <https://en.cppreference.com/w/cpp/language/constexpr>  

---

## Topic Overview

### What Is `constexpr`

`constexpr` marks a function as **potentially evaluable at compile time**. If all arguments are constant expressions, the compiler computes the result at compile time. Otherwise, the function runs normally at runtime.

```cpp

constexpr int square(int x) { return x * x; }

constexpr int ct = square(5);    // Compile-time: ct = 25
int n = get_input();
int rt = square(n);              // Runtime: computed at execution

```

### Evolution of `constexpr` Across Standards

| Standard | What's Allowed in `constexpr` |
| --- | --- |
| C++11 | Single return statement, no loops, no local variables |
| C++14 | Loops, local variables, multiple statements |
| C++17 | `if constexpr`, `constexpr` lambdas |
| C++20 | Virtual calls, `try`, `dynamic_cast`, `std::vector` (transient), `constexpr` algorithms |
| C++23 | `constexpr` `std::unique_ptr` (transient), more library constexpr |

### `constexpr` vs `consteval` vs `const`

| Keyword | Meaning |
| --- | --- |
| `constexpr` | **May** run at compile time or runtime |
| `consteval` | **Must** run at compile time only (C++20) |
| `const` | Value cannot change after initialization (not necessarily compile-time) |

---

## Self-Assessment

### Q1: Write a `constexpr` function and verify it runs at compile time using `static_assert` and at runtime for non-const inputs

```cpp

#include <iostream>
#include <cstdint>

// === constexpr fibonacci ===
constexpr uint64_t fibonacci(int n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;

    uint64_t a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        uint64_t temp = a + b;
        a = b;
        b = temp;
    }
    return b;
}

// === Compile-time verification ===
static_assert(fibonacci(0) == 0);
static_assert(fibonacci(1) == 1);
static_assert(fibonacci(2) == 1);
static_assert(fibonacci(10) == 55);
static_assert(fibonacci(20) == 6765);
static_assert(fibonacci(50) == 12586269025ULL);

// === constexpr power ===
constexpr int power(int base, int exp) {
    int result = 1;
    for (int i = 0; i < exp; ++i)
        result *= base;
    return result;
}

static_assert(power(2, 0) == 1);
static_assert(power(2, 10) == 1024);
static_assert(power(3, 5) == 243);

// === constexpr GCD ===
constexpr int gcd(int a, int b) {
    while (b != 0) {
        int temp = b;
        b = a % b;
        a = temp;
    }
    return a;
}

static_assert(gcd(12, 8) == 4);
static_assert(gcd(100, 75) == 25);

int main() {
    // === Compile-time usage ===
    constexpr auto fib20 = fibonacci(20);
    std::cout << "Compile-time: fibonacci(20) = " << fib20 << "\n";

    // === Runtime usage — same function ===
    int n;
    std::cout << "Enter n for fibonacci: ";
    std::cin >> n;
    std::cout << "Runtime: fibonacci(" << n << ") = " << fibonacci(n) << "\n";

    // === Power at runtime ===
    int base, exp;
    std::cout << "Enter base and exponent: ";
    std::cin >> base >> exp;
    std::cout << "Runtime: power(" << base << ", " << exp << ") = "
              << power(base, exp) << "\n";

    // The same function works in both contexts — that's the point of constexpr
    std::cout << "\nCompile-time: power(2, 10) = " << power(2, 10) << "\n";
    std::cout << "Compile-time: gcd(12, 8) = " << gcd(12, 8) << "\n";

    return 0;
}

```

**Expected output (with input `10`, then `2 8`):**

```text

Compile-time: fibonacci(20) = 6765
Enter n for fibonacci: 10
Runtime: fibonacci(10) = 55
Enter base and exponent: 2 8
Runtime: power(2, 8) = 256

Compile-time: power(2, 10) = 1024
Compile-time: gcd(12, 8) = 4

```

### Q2: Show the limitations of `constexpr` in C++11 vs what became possible in C++14 and C++17

```cpp

#include <iostream>
#include <type_traits>

// === C++11 constexpr: SEVERE restrictions ===
// Only ONE return statement, no loops, no local variables
// Must use recursion for anything complex

// C++11 factorial — recursive, single return
constexpr int factorial_cpp11(int n) {
    return n <= 1 ? 1 : n * factorial_cpp11(n - 1);
}

// C++11 abs — single expression
constexpr int abs_cpp11(int x) {
    return x < 0 ? -x : x;
}

static_assert(factorial_cpp11(5) == 120);
static_assert(abs_cpp11(-42) == 42);

// === C++14 constexpr: loops + local variables ===
// Much more natural — write normal code

constexpr int factorial_cpp14(int n) {
    int result = 1;
    for (int i = 2; i <= n; ++i)
        result *= i;
    return result;
}

constexpr int count_digits(int n) {
    if (n == 0) return 1;
    int count = 0;
    int temp = n < 0 ? -n : n;
    while (temp > 0) {
        ++count;
        temp /= 10;
    }
    return count;
}

static_assert(factorial_cpp14(5) == 120);
static_assert(count_digits(12345) == 5);
static_assert(count_digits(0) == 1);

// === C++17 constexpr: if constexpr + constexpr lambdas ===

template<typename T>
constexpr auto to_string_length(T val) {
    if constexpr (std::is_integral_v<T>) {
        return count_digits(val);
    } else if constexpr (std::is_floating_point_v<T>) {
        return -1; // floating point: variable length
    } else {
        return 0;
    }
}

static_assert(to_string_length(42) == 2);
static_assert(to_string_length(12345) == 5);

// C++17 constexpr lambda
constexpr auto square = [](int x) { return x * x; };
static_assert(square(7) == 49);

int main() {
    std::cout << "=== constexpr Evolution ===\n\n";

    std::cout << "C++11:\n";
    std::cout << "  - Single return statement only\n";
    std::cout << "  - No loops, no local variables\n";
    std::cout << "  - Must use ternary and recursion\n";
    std::cout << "  - factorial_cpp11(5) = " << factorial_cpp11(5) << "\n";

    std::cout << "\nC++14:\n";
    std::cout << "  - Loops, local variables, and multiple statements\n";
    std::cout << "  - Much more natural code\n";
    std::cout << "  - factorial_cpp14(5) = " << factorial_cpp14(5) << "\n";
    std::cout << "  - count_digits(12345) = " << count_digits(12345) << "\n";

    std::cout << "\nC++17:\n";
    std::cout << "  - if constexpr (branch at compile time)\n";
    std::cout << "  - constexpr lambdas\n";
    std::cout << "  - square(7) = " << square(7) << "\n";

    std::cout << "\nC++20:\n";
    std::cout << "  - constexpr std::vector (transient allocation)\n";
    std::cout << "  - constexpr std::string (transient)\n";
    std::cout << "  - constexpr <algorithm> (sort, find, etc.)\n";
    std::cout << "  - constexpr virtual functions\n";
    std::cout << "  - constexpr try/catch (limited)\n";
    std::cout << "  - consteval (must be compile-time)\n";
    std::cout << "  - constinit (must be constant-initialized)\n";

    return 0;
}

```

**Expected output:**

```text

=== constexpr Evolution ===

C++11:

  - Single return statement only
  - No loops, no local variables
  - Must use ternary and recursion
  - factorial_cpp11(5) = 120

C++14:

  - Loops, local variables, and multiple statements
  - Much more natural code
  - factorial_cpp14(5) = 120
  - count_digits(12345) = 5

C++17:

  - if constexpr (branch at compile time)
  - constexpr lambdas
  - square(7) = 49

C++20:

  - constexpr std::vector (transient allocation)
  - constexpr std::string (transient)
  - constexpr <algorithm> (sort, find, etc.)
  - constexpr virtual functions
  - constexpr try/catch (limited)
  - consteval (must be compile-time)
  - constinit (must be constant-initialized)

```

### Q3: Write a `constexpr` sort of a `std::array` using `constexpr` algorithms (C++20)

```cpp

#include <iostream>
#include <array>
#include <algorithm>
#include <numeric>
#include <functional>

// === C++20: std::sort is constexpr! ===

constexpr auto sorted_array() {
    std::array<int, 10> arr = {9, 3, 7, 1, 5, 8, 2, 6, 4, 10};
    std::sort(arr.begin(), arr.end());
    return arr;
}

constexpr auto reversed_array() {
    std::array<int, 10> arr = {9, 3, 7, 1, 5, 8, 2, 6, 4, 10};
    std::sort(arr.begin(), arr.end(), std::greater<>{});
    return arr;
}

constexpr auto unique_sorted() {
    std::array<int, 12> arr = {3, 1, 4, 1, 5, 9, 2, 6, 5, 3, 5, 8};
    std::sort(arr.begin(), arr.end());
    // Note: std::unique requires sorted input
    auto last = std::unique(arr.begin(), arr.end());
    // Count unique elements
    std::size_t count = last - arr.begin();
    return std::pair{arr, count};
}

// All computed at compile time
constexpr auto ascending = sorted_array();
constexpr auto descending = reversed_array();
constexpr auto [unique_arr, unique_count] = unique_sorted();

// Compile-time verification
static_assert(ascending[0] == 1);
static_assert(ascending[9] == 10);
static_assert(descending[0] == 10);
static_assert(descending[9] == 1);
static_assert(unique_count == 8);  // 1,2,3,4,5,6,8,9

// === Compile-time binary search on sorted data ===
constexpr bool ct_contains(int value) {
    return std::binary_search(ascending.begin(), ascending.end(), value);
}

static_assert(ct_contains(5));
static_assert(!ct_contains(11));

// === Compile-time partition ===
constexpr auto partition_evens() {
    std::array<int, 10> arr = {1, 2, 3, 4, 5, 6, 7, 8, 9, 10};
    auto mid = std::partition(arr.begin(), arr.end(),
                               [](int x) { return x % 2 == 0; });
    std::size_t even_count = mid - arr.begin();
    return std::pair{arr, even_count};
}

constexpr auto [partitioned, even_count] = partition_evens();
static_assert(even_count == 5);

int main() {
    std::cout << "=== Constexpr Sort (C++20) ===\n";

    std::cout << "Ascending:  ";
    for (int v : ascending) std::cout << v << " ";
    std::cout << "\n";

    std::cout << "Descending: ";
    for (int v : descending) std::cout << v << " ";
    std::cout << "\n";

    std::cout << "\n=== Unique Sorted ===\n";
    std::cout << "Unique elements (" << unique_count << "): ";
    for (std::size_t i = 0; i < unique_count; ++i)
        std::cout << unique_arr[i] << " ";
    std::cout << "\n";

    std::cout << "\n=== Binary Search (compile-time) ===\n";
    for (int v : {1, 5, 10, 11, 0}) {
        std::cout << "contains(" << v << ") = "
                  << (ct_contains(v) ? "true" : "false") << "\n";
    }

    std::cout << "\n=== Partitioned (evens first) ===\n";
    std::cout << "Even count: " << even_count << "\n";
    std::cout << "Array: ";
    for (int v : partitioned) std::cout << v << " ";
    std::cout << "\n";

    std::cout << "\nAll operations computed at compile time (zero runtime cost).\n";

    return 0;
}

```

**Expected output:**

```text

=== Constexpr Sort (C++20) ===
Ascending:  1 2 3 4 5 6 7 8 9 10
Descending: 10 9 8 7 6 5 4 3 2 1

=== Unique Sorted ===
Unique elements (8): 1 2 3 4 5 6 8 9

=== Binary Search (compile-time) ===
contains(1) = true
contains(5) = true
contains(10) = true
contains(11) = false
contains(0) = false

=== Partitioned (evens first) ===
Even count: 5
Array: 10 2 8 4 6 5 7 3 9 1

All operations computed at compile time (zero runtime cost).

```

---

## Notes

- `constexpr` = CAN run at compile time; `consteval` = MUST run at compile time; `constinit` = must be constant-initialized.
- C++14 removed most C++11 restrictions: loops, local variables, and multiple statements are allowed.
- C++20 made `<algorithm>` constexpr: `sort`, `find`, `binary_search`, `partition`, `unique`, etc.
- `std::vector` and `std::string` can be used inside constexpr functions (transient allocation — must not leak out).
- Use `static_assert(expr)` to prove compile-time evaluation.
- A `constexpr` function called with non-constant arguments runs at runtime — same code, dual behavior.
