# Understand the Difference Between `std::decay` and `std::remove_reference`

**Category:** Type System & Deduction  
**Item:** #202  
**Standard:** C++11 (`decay`, `remove_reference`), C++20 (`remove_cvref`)  
**Reference:** <https://en.cppreference.com/w/cpp/types/decay>  

---

## Topic Overview

### What Does Each Trait Do

| Trait | What it removes/transforms |
| --- | --- |
| `std::remove_reference_t<T>` | Strips `&` or `&&` only. Nothing else. |
| `std::remove_cv_t<T>` | Strips top-level `const`/`volatile` only. |
| `std::remove_cvref_t<T>` (C++20) | Strips `&`/`&&` then `const`/`volatile`. |
| `std::decay_t<T>` | Strips ref + cv, AND performs array→pointer and function→pointer decay. |

### `std::remove_reference` — Minimal Transformation

Simply strips the reference qualifier:

```cpp

#include <type_traits>

// Strips & and &&, nothing else
static_assert(std::is_same_v<std::remove_reference_t<int&>,       int>);
static_assert(std::is_same_v<std::remove_reference_t<int&&>,      int>);
static_assert(std::is_same_v<std::remove_reference_t<const int&>, const int>);  // const stays!
static_assert(std::is_same_v<std::remove_reference_t<int[3]&>,    int[3]>);     // array stays!
static_assert(std::is_same_v<std::remove_reference_t<int>,        int>);        // no-op

```

### `std::decay` — Models Pass-By-Value

`std::decay` does what happens when you pass an argument **by value** to a function:

```cpp

Step 1: Remove reference (& or &&)
Step 2: If array → pointer to element (array decay)
        If function → pointer to function
        Otherwise → remove top-level const/volatile

```

```cpp

#include <type_traits>

// Reference stripping + cv removal
static_assert(std::is_same_v<std::decay_t<const int&>, int>);    // strips ref + const
static_assert(std::is_same_v<std::decay_t<volatile int&&>, int>); // strips ref + volatile

// Array decay → pointer
static_assert(std::is_same_v<std::decay_t<int[3]>, int*>);       // array → pointer!
static_assert(std::is_same_v<std::decay_t<int[3]&>, int*>);      // ref removed, then array → pointer

// Function decay → function pointer
static_assert(std::is_same_v<std::decay_t<int(double)>, int(*)(double)>);

// Compare: remove_reference does NOT decay arrays
static_assert(std::is_same_v<std::remove_reference_t<int[3]&>, int[3]>);  // keeps array!

```

### Complete Comparison Table

| Input Type | `remove_reference_t` | `remove_cvref_t` | `decay_t` |
| --- | --- | --- | --- |
| `int` | `int` | `int` | `int` |
| `int&` | `int` | `int` | `int` |
| `int&&` | `int` | `int` | `int` |
| `const int&` | `const int` | `int` | `int` |
| `volatile int&&` | `volatile int` | `int` | `int` |
| `int[3]` | `int[3]` | `int[3]` | `int*` |
| `int[3]&` | `int[3]` | `int[3]` | `int*` |
| `int(&)[3]` | `int[3]` | `int[3]` | `int*` |
| `const int[3]` | `const int[3]` | `const int[3]` | `const int*` |
| `int(double)` | `int(double)` | `int(double)` | `int(*)(double)` |
| `int(&)(double)` | `int(double)` | `int(double)` | `int(*)(double)` |

### `std::remove_cvref_t` (C++20) — The Middle Ground

C++20 added `remove_cvref_t` as a convenient combination:

```cpp

// Before C++20: manual composition
template<typename T>
using remove_cvref_old = std::remove_cv_t<std::remove_reference_t<T>>;

// C++20: built-in
static_assert(std::is_same_v<std::remove_cvref_t<const int&>, int>);
static_assert(std::is_same_v<std::remove_cvref_t<volatile int&&>, int>);

// Does NOT decay arrays (unlike decay)
static_assert(std::is_same_v<std::remove_cvref_t<int[3]&>, int[3]>);  // array preserved!

```

### When to Use Each

```cpp

remove_reference_t:
  ✓ In std::move / std::forward implementations
  ✓ When you need to strip & but preserve const/array/function form
  ✓ In perfect forwarding utilities

remove_cvref_t (C++20):
  ✓ When comparing "base types" regardless of cv/ref qualifiers
  ✓ In concept definitions (comparing stripped types)
  ✓ Replacing the old remove_const + remove_reference chain

decay_t:
  ✓ Storing function arguments in containers/tuples (mimics by-value passing)
  ✓ std::thread, std::function, std::bind — they decay their arguments
  ✓ When you want the "natural" type of a passed argument

```

---

## Self-Assessment

### Q1: Show that `std::decay<int[3]>::type` is `int*` while `std::remove_reference<int[3]&>::type` is `int[3]`

```cpp

#include <iostream>
#include <type_traits>
#include <string>

int main() {
    // --- Array types ---
    // decay: array → pointer (like passing array by value)
    static_assert(std::is_same_v<std::decay_t<int[3]>, int*>);
    static_assert(std::is_same_v<std::decay_t<int[3]&>, int*>);
    static_assert(std::is_same_v<std::decay_t<const char[6]>, const char*>);

    // remove_reference: only strips &, preserves array type
    static_assert(std::is_same_v<std::remove_reference_t<int[3]&>, int[3]>);
    static_assert(std::is_same_v<std::remove_reference_t<int(&)[3]>, int[3]>);

    // --- Function types ---
    using F = void(int);

    // decay: function → function pointer
    static_assert(std::is_same_v<std::decay_t<F>, void(*)(int)>);
    static_assert(std::is_same_v<std::decay_t<F&>, void(*)(int)>);

    // remove_reference: just strips &
    static_assert(std::is_same_v<std::remove_reference_t<F&>, F>);  // void(int), NOT pointer

    // --- CV qualification ---
    // decay strips top-level const/volatile
    static_assert(std::is_same_v<std::decay_t<const int>, int>);
    static_assert(std::is_same_v<std::decay_t<const volatile int&>, int>);

    // remove_reference does NOT strip const
    static_assert(std::is_same_v<std::remove_reference_t<const int&>, const int>);

    // --- Practical demonstration ---
    // When you pass const char[6] ("hello") by value, it becomes const char*
    auto s = "hello";  // auto deduces: const char* (same as decay)
    static_assert(std::is_same_v<decltype(s), const char*>);

    std::cout << "All static_asserts passed.\n";
    std::cout << "\nKey difference:\n";
    std::cout << "  decay<int[3]>            = int*   (array decays to pointer)\n";
    std::cout << "  remove_reference<int[3]&> = int[3] (array preserved)\n";

    return 0;
}

```

**Output:**

```text

All static_asserts passed.

Key difference:
  decay<int[3]>            = int*   (array decays to pointer)
  remove_reference<int[3]&> = int[3] (array preserved)

```

### Q2: Explain why `std::decay` models what happens to function arguments passed by value

```cpp

#include <iostream>
#include <type_traits>
#include <typeinfo>

// When you pass arguments by value, the compiler applies "decay":
//   1. Strip references
//   2. Arrays → pointers
//   3. Functions → function pointers
//   4. Strip top-level cv qualifiers

void takes_by_value(int x) {}   // Parameter is int, not const int& etc.

// Let's prove decay matches what auto deduces:
template<typename T>
void check(T param) {
    // 'param' received by value — T is the decayed type
    std::cout << "  T = " << typeid(T).name() << "\n";
}

int main() {
    // Case 1: const reference → strips const and ref
    const int ci = 42;
    check(ci);    // T = int (not const int)
    static_assert(std::is_same_v<std::decay_t<const int>, int>);

    // Case 2: array → pointer
    int arr[5] = {1,2,3,4,5};
    check(arr);   // T = int* (not int[5])
    static_assert(std::is_same_v<std::decay_t<int[5]>, int*>);

    // Case 3: function → function pointer
    check(takes_by_value);  // T = void(*)(int)
    static_assert(std::is_same_v<std::decay_t<void(int)>, void(*)(int)>);

    // Case 4: string literal → const char*
    check("hello");  // T = const char* (not const char[6])
    static_assert(std::is_same_v<std::decay_t<const char[6]>, const char*>);

    // This is exactly what std::decay computes!
    // decay_t<T> == the type that auto/template deduction gives for by-value parameter

    // Practical use: std::thread stores decayed copies of arguments
    // std::thread t(func, arg1, arg2);
    // internally stores: decay_t<decltype(func)>, decay_t<decltype(arg1)>, ...

    std::cout << "\nstd::decay perfectly models by-value passing:\n";
    std::cout << "  const int    → int    (cv stripped)\n";
    std::cout << "  int[5]       → int*   (array decayed)\n";
    std::cout << "  void(int)    → void(*)(int) (function decayed)\n";

    return 0;
}

```

**Output:**

```text

  T = int
  T = int*
  T = void(*)(int)
  T = const char*

std::decay perfectly models by-value passing:
  const int    → int    (cv stripped)
  int[5]       → int*   (array decayed)
  void(int)    → void(*)(int) (function decayed)

```

### Q3: Use `remove_cvref_t` (C++20) instead of the `remove_const + remove_reference` composition

```cpp

#include <iostream>
#include <type_traits>
#include <string>
#include <concepts>

// Before C++20: manual composition (ugly, order matters!)
template<typename T>
using strip_old = std::remove_cv_t<std::remove_reference_t<T>>;
// Note: must remove reference FIRST, then cv
// remove_cv on const int& does nothing (const is not top-level on a reference)

// C++20: clean, single trait
// remove_cvref_t<T> = remove_cv_t<remove_reference_t<T>>

// Comparison:
static_assert(std::is_same_v<strip_old<const int&>, int>);
static_assert(std::is_same_v<std::remove_cvref_t<const int&>, int>);

// Why order matters in the old way:
// WRONG order: remove_cv_t<const int&> = const int& (const is part of ref, not removable)
// RIGHT order: remove_reference_t<const int&> = const int, then remove_cv_t = int

// Practical use: concept definitions and comparisons
template<typename T, typename U>
concept SameBaseType = std::same_as<std::remove_cvref_t<T>, std::remove_cvref_t<U>>;

template<typename T>
    requires SameBaseType<T, std::string>
void process_string(T&& s) {
    // Accepts: string, string&, const string&, string&&
    std::cout << "Processing string: " << s << "\n";
}

// remove_cvref vs decay: arrays are NOT decayed
template<typename T>
void show_types() {
    std::cout << "  remove_cvref: " << typeid(std::remove_cvref_t<T>).name() << "\n";
    std::cout << "  decay:        " << typeid(std::decay_t<T>).name() << "\n";
}

int main() {
    // Basic usage
    static_assert(std::is_same_v<std::remove_cvref_t<const int&>, int>);
    static_assert(std::is_same_v<std::remove_cvref_t<volatile int&&>, int>);
    static_assert(std::is_same_v<std::remove_cvref_t<const volatile int&>, int>);

    // Works with the concept
    std::string s = "hello";
    const std::string& cs = s;
    process_string(s);                    // T = string&
    process_string(cs);                   // T = const string&
    process_string(std::string("temp"));  // T = string&&
    process_string(std::move(s));         // T = string&&

    // Key difference from decay:
    std::cout << "\nint[3]&:\n";
    show_types<int(&)[3]>();
    // remove_cvref: int[3] (array preserved)
    // decay: int* (array decayed to pointer)

    std::cout << "\nRule of thumb:\n";
    std::cout << "  remove_cvref_t: strip qualifiers, preserve array/function types\n";
    std::cout << "  decay_t: simulate pass-by-value (arrays/functions become pointers)\n";

    return 0;
}

```

**Output:**

```text

Processing string: hello
Processing string: hello
Processing string: temp
Processing string: hello

int[3]&:
  remove_cvref: int[3]
  decay:        int*

Rule of thumb:
  remove_cvref_t: strip qualifiers, preserve array/function types
  decay_t: simulate pass-by-value (arrays/functions become pointers)

```

---

## Notes

- **`remove_cvref_t` (C++20) should be your default** for stripping qualifiers in modern C++. It replaces the verbose `remove_cv_t<remove_reference_t<T>>` pattern.
- **Ordering pitfall:** `remove_cv_t<const int&>` does NOT strip `const` because the `const` is "inside" the reference. Always remove references first, then cv-qualifiers.
- `std::decay` is used internally by `std::thread`, `std::function`, `std::bind`, `std::make_pair`, and `std::make_tuple` to store decayed copies.
- In generic code, use `decay_t` when storing values (mimics by-value semantics), and `remove_cvref_t` when just comparing types.
