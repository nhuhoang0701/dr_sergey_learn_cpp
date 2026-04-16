# Use nullptr instead of NULL or 0 for null pointers

**Category:** Core Language Fundamentals  
**Item:** #3  
**Standard:** C++11  
**Reference:** <https://github.com/AnthonyCalandra/modern-cpp-features/blob/master/CPP11.md#nullptr>  

---

## Topic Overview

`nullptr` is a keyword introduced in C++11 that represents a null pointer value.
It has the distinct type `std::nullptr_t`, which is NOT an integer type.

### The Problem with NULL and 0

Before C++11, null pointers were represented by `NULL` (a macro typically defined as `0` or `0L`)
or the literal `0`:

```cpp

// In <cstddef> or <stddef.h>:
#define NULL 0    // or #define NULL 0L

```

Since `NULL` is just `0`, it's an **integer**, not a pointer. This causes problems:

```cpp

void process(int value) {
    std::cout << "Called with int: " << value << "\n";
}
void process(int* ptr) {
    std::cout << "Called with pointer: " << ptr << "\n";
}

process(0);       // Calls process(int)! Not the pointer version.
process(NULL);    // AMBIGUOUS! NULL is 0 (int), both overloads match.
                  // Some compilers pick int, some report error.
process(nullptr); // Calls process(int*) — unambiguous!

```

### nullptr Type: std::nullptr_t

```cpp

#include <type_traits>

// nullptr has its own unique type:
static_assert(std::is_same_v<decltype(nullptr), std::nullptr_t>);

// nullptr_t converts to ANY pointer type:
int* p1 = nullptr;        // OK
double* p2 = nullptr;     // OK
std::string* p3 = nullptr; // OK
void* p4 = nullptr;       // OK

// But NOT to integer types:
// int x = nullptr;        // ❌ ERROR: cannot convert nullptr to int
// bool b = nullptr;       // ⚠ May work (contextual conversion to bool)

```

### In Templates

`nullptr` deduces correctly as `std::nullptr_t`, while `NULL` deduces as `int`:

```cpp

template<typename T>
void inspect(T value) {
    if constexpr (std::is_null_pointer_v<T>) {
        std::cout << "null pointer\n";
    } else if constexpr (std::is_integral_v<T>) {
        std::cout << "integer: " << value << "\n";
    } else if constexpr (std::is_pointer_v<T>) {
        std::cout << "pointer\n";
    }
}

inspect(NULL);    // "integer: 0" — T = int (or long)
inspect(nullptr); // "null pointer" — T = std::nullptr_t
inspect(0);       // "integer: 0" — T = int

```

### Forwarding Through Templates

```cpp

template<typename T>
void forward_to_api(T arg) {
    legacy_c_function(arg);  // expects a pointer
}

forward_to_api(NULL);    // T = int → ERROR: int is not a pointer!
forward_to_api(nullptr); // T = nullptr_t → converts to pointer automatically

```

### Using nullptr in Conditions

```cpp

int* ptr = get_optional_result();

// All of these are equivalent:
if (ptr != nullptr) { ... }
if (ptr) { ... }            // implicit conversion to bool

// For smart pointers too:
std::unique_ptr<Widget> up = make_widget();
if (up) { ... }              // checks if not null
if (up != nullptr) { ... }   // same thing, more explicit

```


---

## Self-Assessment

### Q1: Write code where passing NULL to an overloaded function taking int vs pointer causes an ambiguity that nullptr resolves

```cpp

#include <iostream>

// Two overloads: one for int, one for pointer
void process(int value) {
    std::cout << "int overload: " << value << "\n";
}
void process(double* ptr) {
    std::cout << "pointer overload: " << ptr << "\n";
}

int main() {
    // process(NULL);    // ❌ Ambiguous: NULL is 0 (int literal)
    //                   // Both process(int) and process(double*) match
    //                   // Compiler error: "ambiguous overload"

    process(nullptr);    // ✅ "pointer overload: 0"
    //                   // nullptr is std::nullptr_t → converts to double*
    //                   // Cannot convert to int → only one match

    process(0);          // "int overload: 0"
    //                   // 0 is an int — matches process(int) directly
}

```

The ambiguity with `NULL` occurs because `0` is both a valid `int` and converts to any pointer type
via the null pointer conversion. The compiler can't choose between two equally-good conversions.
`nullptr` eliminates this because `std::nullptr_t` converts to pointers but NOT to `int`.


### Q2: Show how nullptr has type std::nullptr_t and can be used in templates

```cpp

#include <iostream>
#include <type_traits>
#include <cstddef>

// nullptr has type std::nullptr_t
static_assert(std::is_same_v<decltype(nullptr), std::nullptr_t>);
static_assert(std::is_null_pointer_v<decltype(nullptr)>);

// std::nullptr_t can be used in templates:
template<typename T>
std::string type_name(T) {
    if constexpr (std::is_null_pointer_v<T>)
        return "nullptr_t";
    else if constexpr (std::is_pointer_v<T>)
        return "pointer";
    else if constexpr (std::is_integral_v<T>)
        return "integer";
    else
        return "other";
}

int main() {
    std::cout << type_name(nullptr) << "\n";  // "nullptr_t"
    std::cout << type_name(0) << "\n";         // "integer"
    std::cout << type_name((int*)0) << "\n";   // "pointer"

    // You can have a function that specifically accepts nullptr:
    auto only_null = [](std::nullptr_t) {
        std::cout << "received nullptr\n";
    };
    only_null(nullptr);  // OK
    // only_null(0);      // ❌ ERROR: int is not nullptr_t
}

```


### Q3: Demonstrate a compile error that NULL produces but nullptr does not when deducing template type

```cpp

#include <iostream>

// Template that expects to receive a pointer:
template<typename T>
void call_with_ptr(T ptr) {
    // Try to dereference — only works if T is actually a pointer
    if (ptr != nullptr) {  // ERROR if T = int (from NULL)
        std::cout << *ptr << "\n";
    }
}

template<typename T>
void pass_to_api(T arg) {
    int* destination = arg;  // only works if T converts to int*
    // If T = int (from NULL), this is int* = int → ERROR
}

int main() {
    int value = 42;

    // With NULL:
    // call_with_ptr(NULL);
    // T deduces to int (or long), then:
    //   - "ptr != nullptr" → comparing int with nullptr_t → ERROR
    //   - "*ptr" → dereferencing int → ERROR

    // pass_to_api(NULL);
    // T = int → "int* destination = int_value" → ERROR

    // With nullptr — everything works:
    call_with_ptr(&value);   // T = int*, works perfectly
    call_with_ptr(nullptr);  // T = std::nullptr_t, nullptr != nullptr is false, OK
    pass_to_api(&value);     // T = int*, works perfectly
    pass_to_api(nullptr);    // T = std::nullptr_t, converts to int*, OK
}

```

The key difference: `NULL` causes `T` to deduce as an integer type, which then fails to work
in pointer contexts. `nullptr` deduces as `std::nullptr_t`, which correctly converts to any pointer.


---

## Notes

- `nullptr` is a keyword, not a macro — it's always available without any `#include`.
- `std::nullptr_t` is implicitly convertible to any pointer type and any pointer-to-member type.
- In template code, `NULL` deduces as `int` (or `long`) while `nullptr` deduces as `std::nullptr_t` — this is the most common source of template errors with `NULL`.
- `nullptr` can be contextually converted to `bool` (evaluates to `false`), but cannot be implicitly converted to integer types.
- C++ Core Guidelines: ES.47 — "Use nullptr rather than 0 or NULL."
- Some legacy C APIs accept `0` for null — you can still pass `nullptr` to them since it converts to any pointer type.
