# Know how user-defined literals work and define your own

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/user_literal>  

---

## Topic Overview

### What Are User-Defined Literals

User-defined literals (UDLs) let you bolt a custom *suffix* onto a literal value, so a bare number or string can carry meaning and type with it. Instead of an anonymous `5.0`, you write `5.0_km` and get back an actual `Kilometers` object:

```cpp
auto distance = 5.0_km;     // Returns a Kilometers object
auto greeting = "hello"_s;   // Returns a std::string
auto angle = 90_deg;         // Returns a Radians value
```

The payoff is readable, type-safe code: the unit lives in the value, not in a comment.

### Syntax

You create a UDL by defining a specially-named `operator""` function. Which signature you write depends on what kind of literal you want to attach the suffix to:

```cpp
// For integer literals: 42_suffix
ReturnType operator""_suffix(unsigned long long value);

// For floating-point literals: 3.14_suffix
ReturnType operator""_suffix(long double value);

// For string literals: "hello"_suffix
ReturnType operator""_suffix(const char* str, std::size_t len);

// For character literals: 'A'_suffix
ReturnType operator""_suffix(char c);

// Raw form (receives digits as string): 123_suffix
ReturnType operator""_suffix(const char* raw_digits);

// Template form (compile-time): 123_suffix
template<char... Digits>
ReturnType operator""_suffix();
```

A practical note: for numeric suffixes you usually want *both* the `unsigned long long` and `long double` overloads, so that `42_km` and `42.0_km` both compile.

### Suffix Naming Rules

There's one rule you must never forget, and a couple of corollaries:

1. **Your suffixes MUST start with `_`** - suffixes without a leading underscore are reserved for the standard library.
2. A `_` followed by an uppercase letter is reserved for the *implementation* (the compiler/stdlib).
3. So: `_km` is good, `_m` is good, `km` is forbidden (reserved), `_Km` is forbidden (reserved for the implementation).

### Standard Library Literals

The reason suffixes-without-underscore are reserved is that the standard library uses them. They live in inline namespaces you opt into:

```cpp
using namespace std::literals;

auto s = "hello"s;           // std::string (std::string_literals)
auto sv = "hello"sv;         // std::string_view (std::string_view_literals)
auto dur = 100ms;            // std::chrono::milliseconds (std::chrono_literals)
auto cx = 3.0 + 4.0i;       // std::complex<double> (std::complex_literals)
```

---

## Self-Assessment

### Q1: Define `operator""_km` and `operator""_m` to create type-safe distance values

This is the canonical "strong units" example. The whole point is that `Kilometers` and `Meters` are *separate types*, so the compiler refuses to let you mix them by accident:

```cpp
#include <iostream>
#include <compare>

// Strong type for distance - prevents mixing units
struct Kilometers {
    double value;
    auto operator<=>(const Kilometers&) const = default;
};

struct Meters {
    double value;
    auto operator<=>(const Meters&) const = default;
};

// User-defined literal operators
Kilometers operator""_km(long double val) {
    return Kilometers{static_cast<double>(val)};
}

Kilometers operator""_km(unsigned long long val) {
    return Kilometers{static_cast<double>(val)};
}

Meters operator""_m(long double val) {
    return Meters{static_cast<double>(val)};
}

Meters operator""_m(unsigned long long val) {
    return Meters{static_cast<double>(val)};
}

// Explicit conversion functions
Meters to_meters(Kilometers km) { return Meters{km.value * 1000.0}; }
Kilometers to_km(Meters m) { return Kilometers{m.value / 1000.0}; }

// Arithmetic within the same unit
Kilometers operator+(Kilometers a, Kilometers b) { return {a.value + b.value}; }
Meters operator+(Meters a, Meters b) { return {a.value + b.value}; }

// Stream output
std::ostream& operator<<(std::ostream& os, Kilometers k) { return os << k.value << " km"; }
std::ostream& operator<<(std::ostream& os, Meters m) { return os << m.value << " m"; }

int main() {
    auto marathon = 42.195_km;
    auto sprint = 100_m;
    auto relay = 400_m;

    std::cout << "Marathon: " << marathon << "\n";           // 42.195 km
    std::cout << "Sprint: " << sprint << "\n";               // 100 m
    std::cout << "Sprint + Relay: " << sprint + relay << "\n"; // 500 m

    // Convert to compare:
    auto marathon_m = to_meters(marathon);
    std::cout << "Marathon in meters: " << marathon_m << "\n"; // 42195 m

    // Same-unit addition works:
    auto total = 10.0_km + 5.0_km;
    std::cout << "Total: " << total << "\n";  // 15 km

    return 0;
}
```

**How this works:**

- The `long double` overload catches floating-point literals like `42.195_km`; the `unsigned long long` overload catches integer ones like `100_km`. You need both.
- Because `Kilometers` and `Meters` are **distinct types**, the type system itself blocks accidental mixing.
- Arithmetic is only defined within a single unit, so `km + km` works but `km + m` does not.
- When you genuinely want to cross units, you do it deliberately through the conversion functions.

### Q2: Show a compile error when kilometers and meters are mixed without conversion

Here's the safety net in action - leave out a conversion and the code simply won't build:

```cpp
#include <iostream>

struct Kilometers { double value; };
struct Meters { double value; };

Kilometers operator""_km(long double v) { return {static_cast<double>(v)}; }
Meters operator""_m(long double v) { return {static_cast<double>(v)}; }

// Only same-unit addition is defined:
Kilometers operator+(Kilometers a, Kilometers b) { return {a.value + b.value}; }
Meters operator+(Meters a, Meters b) { return {a.value + b.value}; }

int main() {
    auto dist1 = 5.0_km;
    auto dist2 = 500.0_m;

    // COMPILE ERROR: no matching operator+ for Kilometers + Meters
    // auto bad = dist1 + dist2;
    // error: no match for 'operator+' (operand types are 'Kilometers' and 'Meters')

    // CORRECT: explicit conversion first
    // auto good = dist1 + to_km(dist2);  // Both are Kilometers

    return 0;
}
```

That refusal to compile is the *feature*. Mixing kilometers and meters is exactly the kind of bug that has crashed real spacecraft - strong typing with UDLs turns it from a silent runtime error into a compile error you can't miss.

### Q3: Explain the restrictions on user-defined literal suffixes

Pulling the rules together in one place:

1. **Must start with `_`:** `_km`, `_deg`, `_px` are all fine.
   - Suffixes *without* `_` belong to the standard: `s`, `sv`, `ms`, `i`, and so on.
   - Using a reserved suffix is **ill-formed, no diagnostic required** - your compiler might accept it today, but a future standard could break you with no warning.

2. **Must not be `_` followed by uppercase:** `_Km`, `_DEG` are reserved for the implementation.

3. **Must not contain a double underscore:** `__km` is reserved for the implementation.

4. **The parameter types are a fixed menu:**
   - `unsigned long long` - integer literals
   - `long double` - floating-point literals
   - `const char*, std::size_t` - string literals
   - `char` - character literals
   - `const char*` - the raw literal operator
   - `template<char...>` - the raw literal operator template

5. **UDLs can't be ordinary templates** (the only template form is the `template<char...>` raw numeric form).

6. **UDLs must be non-member or friend functions** - never member functions.

```cpp
// VALID:
long double operator""_deg(long double d) { return d * 3.14159265358979 / 180.0; }

// INVALID (reserved - no underscore):
// long double operator""deg(long double d);  // ERROR or reserved

// INVALID (reserved - underscore + uppercase):
// long double operator""_Deg(long double d);  // Reserved for implementation
```

---

## Additional Examples

### Compile-Time UDL with Template Parameter Pack

The `template<char...>` form gets the literal's digits as individual characters at compile time, so you can fold them into a value entirely during compilation - here, a binary literal:

```cpp
#include <iostream>
#include <cstdint>

// Compile-time binary literal: 1010_b -> 10
template<char... Digits>
constexpr uint64_t operator""_b() {
    static_assert(((Digits == '0' || Digits == '1') && ...), "Binary digits only!");
    uint64_t result = 0;
    ((result = result * 2 + (Digits - '0')), ...);
    return result;
}

int main() {
    constexpr auto val = 11010110_b;  // Compile-time binary literal
    static_assert(val == 214);
    std::cout << "11010110 binary = " << val << "\n";  // 214

    constexpr auto byte = 11111111_b;
    static_assert(byte == 255);
}
```

Note the `static_assert` inside: a non-binary digit fails to *compile*, not at runtime.

### String UDL for Regex

The string form receives both the pointer and the length, which is exactly what `std::regex` wants - so `"..."_re` reads as "this string literal *is* a regex":

```cpp
#include <regex>
#include <iostream>
#include <string>

std::regex operator""_re(const char* pattern, std::size_t len) {
    return std::regex(pattern, len);
}

int main() {
    auto email_pattern = R"([a-zA-Z0-9._%+-]+@[a-zA-Z0-9.-]+\.[a-zA-Z]{2,})"_re;

    std::string test = "user@example.com";
    if (std::regex_match(test, email_pattern)) {
        std::cout << test << " is a valid email\n";
    }
}
```

### Chrono-style Duration UDLs

The chrono literals are the standard library's own UDLs, and they're worth studying because they show how nicely typed durations compose and compare across units:

```cpp
#include <iostream>
#include <chrono>

using namespace std::chrono_literals;

int main() {
    auto timeout = 500ms;           // std::chrono::milliseconds
    auto interval = 2s;             // std::chrono::seconds
    auto precise = 1.5s;            // std::chrono::duration<double>
    auto frame_time = 16'667us;     // std::chrono::microseconds

    auto total = timeout + interval; // 2500ms
    std::cout << "Total: " << total.count() << "ms\n";  // 2500

    // Comparison works across units:
    if (timeout < 1s) {
        std::cout << "Timeout is less than 1 second\n";
    }
}
```

---

## Notes

- UDL suffixes must start with `_` - no underscore means it's reserved for the standard library.
- Provide both the `unsigned long long` and `long double` overloads so `42_km` and `42.0_km` both work.
- UDLs give you **strong typing** that catches unit-mixing bugs at compile time.
- The standard library ships `""s`, `""sv`, `""ms`, `""h`, `""i`, and more.
- Use `constexpr` UDLs for compile-time work like binary literals or hex parsing.
