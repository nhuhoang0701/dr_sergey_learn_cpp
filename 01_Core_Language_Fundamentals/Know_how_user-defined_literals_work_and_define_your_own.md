# Know how user-defined literals work and define your own

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/user_literal>  

---

## Topic Overview

### What Are User-Defined Literals

User-defined literals (UDLs) let you create custom suffixes for literal values, enabling expressive, type-safe syntax like `42_km`, `"hello"_s`, or `3.14_deg`.

```cpp

auto distance = 5.0_km;     // Returns a Kilometers object
auto greeting = "hello"_s;   // Returns a std::string
auto angle = 90_deg;         // Returns a Radians value

```

### Syntax

UDLs are defined as `operator""` functions:

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

### Suffix Naming Rules

1. **User suffixes MUST start with `_` (underscore)** — suffixes without `_` are reserved for the standard library.
2. Suffixes starting with `_` followed by an uppercase letter are reserved for implementations.
3. Examples: `_km` ✅, `_m` ✅, `km` ❌ (reserved), `_Km` ❌ (reserved for impl)

### Standard Library Literals

The standard library provides these (in inline namespaces):

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

```cpp

#include <iostream>
#include <compare>

// Strong type for distance — prevents mixing units
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

- `operator""_km(long double)` handles floating-point literals like `42.195_km`.
- `operator""_km(unsigned long long)` handles integer literals like `100_km`.
- `Kilometers` and `Meters` are **distinct types** — the compiler prevents accidental mixing.
- Arithmetic is only defined within the same unit type.
- Explicit conversion functions enable intentional unit conversion.

### Q2: Show a compile error when kilometers and meters are mixed without conversion

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

**The compiler enforces type safety** — since `Kilometers` and `Meters` are different types, you cannot accidentally add them without an explicit conversion. This is the power of strong typing with UDLs.

### Q3: Explain the restrictions on user-defined literal suffixes

**Rules for UDL suffix names:**

1. **Must start with `_` (underscore):** `_km`, `_deg`, `_px` — all valid.
   - Suffixes WITHOUT `_` are reserved for the C++ standard: `s` (string), `sv` (string_view), `ms` (milliseconds), `i` (complex), etc.
   - Using reserved suffixes is **ill-formed; no diagnostic required** — the compiler might accept it but behavior is undefined in future standards.

2. **Must NOT start with `_` followed by uppercase:** `_Km`, `_DEG` — reserved for the implementation (compiler/standard library).

3. **Must NOT contain double underscore:** `__km` — reserved for implementation.

4. **Allowed parameter types are fixed:**

   - `unsigned long long` — for integer literals
   - `long double` — for floating-point literals
   - `const char*, std::size_t` — for string literals
   - `char` — for character literals
   - `const char*` — raw literal operator
   - Template `<char...>` — raw literal operator template

5. **UDLs cannot be templates** (except the raw `template<char...>` form for numeric literals).

6. **UDLs must be non-member functions or friend functions** — they cannot be member functions.

```cpp

// VALID:
long double operator""_deg(long double d) { return d * 3.14159265358979 / 180.0; }

// INVALID (reserved — no underscore):
// long double operator""deg(long double d);  // ERROR or reserved

// INVALID (reserved — underscore + uppercase):
// long double operator""_Deg(long double d);  // Reserved for implementation

```

---

## Additional Examples

### Compile-Time UDL with Template Parameter Pack

```cpp

#include <iostream>
#include <cstdint>

// Compile-time binary literal: 1010_b → 10
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

### String UDL for Regex

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

- UDL suffixes must start with `_` — no underscore = reserved for the standard library.
- Provide both `unsigned long long` and `long double` overloads to handle `42_km` and `42.0_km`.
- UDLs enable **strong typing** that catches unit-mixing bugs at compile time.
- The standard library provides `""s`, `""sv`, `""ms`, `""h`, `""i` etc.
- Use `constexpr` UDLs for compile-time computation (e.g., binary literals, hex parsing).
