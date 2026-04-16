# Use std::string_view to avoid unnecessary string copies in APIs

**Category:** Standard Library — Containers  
**Item:** #66  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string_view>  

---

## Topic Overview

`std::string_view` (C++17) is a non-owning, lightweight reference to a contiguous sequence of characters. It has the same read-only interface as `std::string` but **never allocates memory**. It's made up of just a pointer and a size — 16 bytes on 64-bit systems.

### Why string_view

```cpp

                       const std::string&          std::string_view
                       ─────────────────           ─────────────────
Accepts std::string    ✅                          ✅
Accepts "literal"      ✅ (constructs temp string!) ✅ (no copy, no alloc)
Accepts string_view    ❌ (needs .data() hack)     ✅
Accepts substring      ❌ (needs substr → alloc)   ✅ (just adjust ptr+len)
Memory allocation      Potentially                  Never

```

### Core API

```cpp

#include <string_view>
#include <string>
#include <iostream>

// === Use string_view in function parameters ===
void print_name(std::string_view name) {    // no copy, no alloc
    std::cout << "Name: " << name << "\n";
    std::cout << "Length: " << name.size() << "\n";
    std::cout << "First: " << name.front() << "\n";
}

int main() {
    std::string s = "Alice";
    const char* c = "Bob";
    std::string_view sv = "Charlie";

    print_name(s);        // no copy (implicit conversion)
    print_name(c);        // no copy (implicit conversion)
    print_name(sv);       // no copy
    print_name("David");  // no copy (no temp string created!)

    // === Substring without allocation ===
    std::string_view full = "Hello, World!";
    std::string_view hello = full.substr(0, 5);     // just ptr+len, no alloc
    std::string_view world = full.substr(7, 5);     // just ptr+len, no alloc
    // Compare: std::string::substr ALWAYS allocates a new string

    // === Trimming operations (no allocation) ===
    std::string_view padded = "   trimmed   ";
    padded.remove_prefix(3);   // modifies the VIEW, not the data
    padded.remove_suffix(3);
    std::cout << "[" << padded << "]\n";  // [trimmed]

    return 0;
}

```

### Key Rules

| Rule | Details |
| --- | --- |
| **Non-owning** | Does not manage lifetime of underlying data |
| **Read-only** | No `operator[]` modification, no `push_back`, etc. |
| **Not null-terminated** | `data()` may not be null-terminated (e.g., after `remove_suffix`) |
| **Implicit from string** | `std::string` → `string_view` is implicit |
| **Explicit to string** | `string_view` → `std::string` requires explicit constructor |

---

## Self-Assessment

### Q1: Rewrite a function taking const std::string& to take std::string_view and verify no allocation occurs

```cpp

#include <iostream>
#include <string>
#include <string_view>

// === BEFORE: const std::string& ===
// This forces allocation when called with a string literal:
//   greet_old("World");  // constructs a temporary std::string → heap allocation!
std::string greet_old(const std::string& name) {
    return "Hello, " + name + "!";
}

// === AFTER: std::string_view ===
// No temporary string created for ANY caller type
std::string greet_new(std::string_view name) {
    std::string result;
    result.reserve(7 + name.size() + 1);  // "Hello, " + name + "!"
    result += "Hello, ";
    result += name;     // string_view appends to string without allocation
    result += "!";
    return result;
}

// === Counting allocations (simplified) ===
struct AllocCounter {
    static inline int count = 0;
};

void* operator new(std::size_t sz) {
    AllocCounter::count++;
    return std::malloc(sz);
}

void operator delete(void* ptr) noexcept { std::free(ptr); }
void operator delete(void* ptr, std::size_t) noexcept { std::free(ptr); }

int main() {
    // Test with string literal
    AllocCounter::count = 0;
    auto r1 = greet_old("World");
    int old_allocs = AllocCounter::count;

    AllocCounter::count = 0;
    auto r2 = greet_new("World");
    int new_allocs = AllocCounter::count;

    std::cout << "greet_old(\"World\"): " << old_allocs << " allocations\n";
    std::cout << "greet_new(\"World\"): " << new_allocs << " allocations\n";
    // The old version needs an extra allocation for the temporary std::string parameter
    // The new version avoids it entirely

    // Test with std::string
    std::string name = "Alice";
    AllocCounter::count = 0;
    auto r3 = greet_old(name);
    old_allocs = AllocCounter::count;

    AllocCounter::count = 0;
    auto r4 = greet_new(name);
    new_allocs = AllocCounter::count;

    std::cout << "greet_old(string):  " << old_allocs << " allocations\n";
    std::cout << "greet_new(string):  " << new_allocs << " allocations\n";
    // With std::string argument, both are similar — no temporary needed

    // Test with substring (biggest win)
    std::string full = "Hello World!!!!";
    std::string_view sv = std::string_view(full).substr(6, 5); // "World" — no alloc

    AllocCounter::count = 0;
    auto r5 = greet_new(sv);
    std::cout << "greet_new(substr_view): " << AllocCounter::count << " allocations\n";
    // Only allocates for the result string, never for the parameter

    return 0;
}

```

**How it works:**

- `const std::string&` forces a temporary `std::string` construction when passed a `const char*` or string literal → heap allocation.
- `std::string_view` accepts all string-like types without any copying or allocation.
- The biggest savings come when passing substrings: `string::substr()` allocates; `string_view::substr()` doesn't.
- **Rule of thumb:** Use `string_view` for function parameters that only READ the string. Use `const string&` only if the function needs to store/own the string.

### Q2: Show the danger of storing a string_view beyond the lifetime of its source string

```cpp

#include <iostream>
#include <string>
#include <string_view>
#include <vector>

// === DANGEROUS: storing string_view as a member ===
class Logger {
    std::string_view name_;  // ⚠️ DANGER: non-owning!
public:
    Logger(std::string_view name) : name_(name) {}
    void log(std::string_view msg) {
        std::cout << "[" << name_ << "] " << msg << "\n";
    }
};

std::string_view dangerous_function() {
    std::string local = "temporary";
    return local;  // ⚠️ DANGLING: local destroyed, view points to freed memory
}

std::string_view also_dangerous() {
    return std::string("also temporary");  // ⚠️ temporary destroyed at semicolon
}

int main() {
    // === Bug 1: dangling from local variable ===
    std::string_view sv = dangerous_function();
    // sv.data() points to destroyed stack memory!
    // std::cout << sv << "\n";  // UNDEFINED BEHAVIOR

    // === Bug 2: dangling from temporary ===
    std::string_view sv2 = also_dangerous();
    // std::cout << sv2 << "\n";  // UNDEFINED BEHAVIOR

    // === Bug 3: dangling from container reallocation ===
    std::vector<std::string> names = {"Alice"};
    std::string_view first = names[0];  // points to names[0]'s buffer
    names.push_back("Bob");             // may reallocate → names[0] moves!
    // first now dangles if reallocation happened
    // std::cout << first << "\n";  // UNDEFINED BEHAVIOR (maybe)

    // === Bug 4: storing string_view in a class ===
    Logger* logger;
    {
        std::string service_name = "AuthService";
        logger = new Logger(service_name);
    }  // service_name destroyed here
    // logger->log("user login");  // UNDEFINED BEHAVIOR: name_ dangles

    // === SAFE patterns ===
    // 1. Use string_view only for parameters (transient use):
    auto print = [](std::string_view s) { std::cout << s << "\n"; };
    print("safe");  // string literal lives for program duration

    // 2. Store std::string in classes, accept string_view in constructors:
    // class SafeLogger {
    //     std::string name_;  // OWNING
    //     SafeLogger(std::string_view name) : name_(name) {}  // copies into string
    // };

    // 3. string_view from string literals is always safe:
    std::string_view safe_sv = "literals live forever";  // OK

    // 4. Convert to string when you need ownership:
    std::string owned(sv);  // explicit conversion copies the data

    delete logger;
    return 0;
}

```

**How it works:**

- `string_view` does NOT own memory. When the source string is destroyed, the view **dangles** — accessing it is undefined behavior.
- Common traps: returning `string_view` to a local, storing in a member variable, holding a view across container reallocation.
- **Safe rule:** Use `string_view` only for **transient** use — function parameters and local computations. For storage, use `std::string`.
- String literals have static lifetime, so `string_view` to a literal is always safe.

### Q3: Explain why string_view::data() is not necessarily null-terminated

```cpp

#include <iostream>
#include <string>
#include <string_view>
#include <cstring>

void c_api_function(const char* str) {
    // Expects null-terminated C-string
    std::cout << "C API: " << str << " (len=" << std::strlen(str) << ")\n";
}

int main() {
    // === Case 1: string_view from string literal — IS null-terminated ===
    std::string_view sv1 = "Hello, World!";
    std::cout << "sv1.data(): " << sv1.data() << "\n";  // OK: underlying literal is null-terminated
    c_api_function(sv1.data());  // Happens to work, but not guaranteed!

    // === Case 2: string_view after remove_suffix — NOT null-terminated ===
    std::string_view sv2 = "Hello, World!";
    sv2.remove_suffix(8);  // sv2 is now "Hello" (5 chars)
    std::cout << "sv2: " << sv2 << "\n";       // prints "Hello" (operator<< respects size)
    std::cout << "sv2.data(): " << sv2.data() << "\n";  // prints "Hello, World!" ← STILL reads past the view!
    std::cout << "sv2.size(): " << sv2.size() << "\n";  // 5

    // c_api_function(sv2.data());  // ⚠️ WRONG: C function reads past the 5-char view

    // === Case 3: string_view from substr — NOT null-terminated ===
    std::string_view sv3 = std::string_view("Hello, World!").substr(0, 5);
    // sv3.data() points to 'H', but there's no '\0' after 'o'
    // The 'o' is followed by ',' in memory
    std::cout << "sv3.data() raw: " << sv3.data() << "\n";  // "Hello, World!" — reads too far!

    // === Case 4: string_view from arbitrary memory ===
    char buffer[] = {'A', 'B', 'C'};  // no null terminator!
    std::string_view sv4(buffer, 3);
    std::cout << "sv4: " << sv4 << "\n";  // "ABC" (safe: operator<< uses size)
    // c_api_function(sv4.data());  // ⚠️ UNDEFINED BEHAVIOR: no null terminator

    // === Safe way to pass string_view to C APIs ===
    // Convert to std::string (which IS null-terminated)
    std::string safe_copy(sv2);  // copies data + adds '\0'
    c_api_function(safe_copy.c_str());  // ✅ SAFE

    // === Why this happens ===
    // string_view is {pointer, size}. It has NO control over what comes after size bytes.
    // remove_prefix/remove_suffix only adjust the pointer/size — they don't insert '\0'.
    // substr creates a new view with different ptr/size — no null terminator inserted.
    //
    // std::string ALWAYS maintains a null terminator.
    // string_view NEVER guarantees one.
    //
    // This is the fundamental trade-off: string_view avoids allocation at the cost of
    // not being compatible with C APIs that expect null-terminated strings.

    // === Summary ===
    std::cout << "\n=== Rules ===\n";
    std::cout << "1. NEVER pass string_view::data() to C APIs unless you know it's null-terminated\n";
    std::cout << "2. Use string_view with size-aware APIs (cout, string ctor, algorithms)\n";
    std::cout << "3. Convert to std::string when you need null termination\n";
    std::cout << "4. string_view from a literal IS null-terminated (but don't rely on it after trimming)\n";

    return 0;
}

```

**How it works:**

- `string_view` is `{const char* data, size_t size}` — it does NOT store or maintain a null terminator.
- When created from a string literal or `std::string`, the underlying data happens to be null-terminated. But `remove_prefix()`, `remove_suffix()`, and `substr()` change the view without inserting `'\0'`.
- **C APIs** (`printf`, `fopen`, `strlen`) read until `'\0'` — passing `string_view::data()` is dangerous unless you verified null termination.
- **C++ APIs** (streams, `std::string` constructor, algorithms) use the `size()` and are safe.
- **Safe conversion:** `std::string(sv)` explicitly copies and adds a null terminator.

---

## Notes

- **Use `string_view` for parameters**, `std::string` for data members and return values that outlive the function.
- **`std::string_view` literal:** `"hello"sv` (C++17, `using namespace std::string_view_literals`).
- **`constexpr`-friendly:** `string_view` operations are `constexpr` — usable in compile-time contexts.
- **C++20 additions:** `starts_with()`, `ends_with()` added to `string_view`. C++23 adds `contains()`.
- **Performance:** `string_view` is typically passed by value (16 bytes on 64-bit) — cheaper than `const string&` (which is a pointer + indirection to heap).
- **Pitfall with `operator+`:** You can't do `sv + " suffix"` because `string_view` has no `operator+`. Convert first: `std::string(sv) + " suffix"`.
- **`std::string::operator string_view()`** is implicit. The reverse is explicit (`std::string(sv)`).
