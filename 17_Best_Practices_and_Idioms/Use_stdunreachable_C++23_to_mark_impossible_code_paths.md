# Use std::unreachable() (C++23) to mark impossible code paths

**Category:** Best Practices & Idioms  
**Item:** #170  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/unreachable>  

---

## Topic Overview

`std::unreachable()` tells the compiler that a code path will never be executed. The compiler can use this to optimize, but reaching it is **undefined behavior**.

```cpp

#include <utility>
switch (color) {
    case Red:   return "#FF0000";
    case Green: return "#00FF00";
    case Blue:  return "#0000FF";
}
std::unreachable();  // "trust me, all cases are covered"

```

---

## Self-Assessment

### Q1: Add `std::unreachable()` to a switch default and verify optimization

```cpp

#include <iostream>
#include <utility>  // std::unreachable (C++23)

enum class Direction { North, South, East, West };

// WITHOUT unreachable: compiler generates code for "default" case
int dx_safe(Direction d) {
    switch (d) {
        case Direction::North: return 0;
        case Direction::South: return 0;
        case Direction::East:  return 1;
        case Direction::West:  return -1;
        default: return 0;  // dead code, but compiler doesn't know
    }
}

// WITH unreachable: compiler knows default can't happen
int dx_fast(Direction d) {
    switch (d) {
        case Direction::North: return 0;
        case Direction::South: return 0;
        case Direction::East:  return 1;
        case Direction::West:  return -1;
    }
    std::unreachable();  // UB if reached, but allows better codegen
}

int main() {
    std::cout << "East dx: " << dx_fast(Direction::East) << '\n';
    std::cout << "West dx: " << dx_fast(Direction::West) << '\n';
    std::cout << "North dx: " << dx_fast(Direction::North) << '\n';
}
// Expected output:
// East dx: 1
// West dx: -1
// North dx: 0

```

**Optimization effect:** Without `unreachable()`, the compiler may generate a branch for the "after switch" path. With it, the compiler eliminates that branch entirely.

### Q2: UB semantics and security implications

`std::unreachable()` invokes **undefined behavior** if actually reached. This means:

```cpp

// DANGER: if the invariant is wrong, ANYTHING can happen!
int get_value(int index) {
    // Assume index is 0, 1, or 2
    switch (index) {
        case 0: return 10;
        case 1: return 20;
        case 2: return 30;
    }
    std::unreachable();
    // If index==3: UB!
    // Compiler may: skip the switch, return garbage, crash, etc.
}

```

**Security implications:**

| Risk | Explanation |
| --- | --- |
| Skipped bounds check | Compiler may remove safety checks after unreachable |
| Memory corruption | UB can lead to arbitrary memory access |
| Exploitation | Attackers can craft inputs that reach "unreachable" paths |

**Safe pattern:**

```cpp

int get_value_safe(int index) {
    switch (index) {
        case 0: return 10;
        case 1: return 20;
        case 2: return 30;
    }
    #ifdef NDEBUG
        std::unreachable();  // release: trust the invariant
    #else
        assert(false && "should not reach here");  // debug: catch violations
    #endif
}

```

### Q3: Compare `std::unreachable` with alternatives

```cpp

#include <iostream>
#include <cassert>
// #include <utility>  // for std::unreachable (C++23)

enum class Color { Red, Green, Blue };

// Option 1: __builtin_unreachable() — GCC/Clang extension
const char* to_string_v1(Color c) {
    switch (c) {
        case Color::Red:   return "red";
        case Color::Green: return "green";
        case Color::Blue:  return "blue";
    }
    __builtin_unreachable();  // GCC/Clang only
}

// Option 2: assert(false) — debug check, no optimization hint
const char* to_string_v2(Color c) {
    switch (c) {
        case Color::Red:   return "red";
        case Color::Green: return "green";
        case Color::Blue:  return "blue";
    }
    assert(false && "impossible color");  // stripped in release
    return "unknown";
}

// Option 3: std::unreachable() — C++23, portable, optimizer hint
// const char* to_string_v3(Color c) {
//     switch (c) { ... }
//     std::unreachable();
// }

int main() {
    std::cout << to_string_v1(Color::Red) << '\n';
    std::cout << to_string_v2(Color::Green) << '\n';
}
// Expected output:
// red
// green

```

| Feature | `std::unreachable` | `__builtin_unreachable` | `assert(false)` |
| --- | --- | --- | --- |
| Standard | C++23 | GCC/Clang extension | C89 |
| Portable | Yes | No (MSVC: `__assume(0)`) | Yes |
| Optimizer hint | Yes | Yes | No |
| Debug check | No | No | Yes |
| UB if reached | Yes | Yes | `abort()` in debug |

---

## Notes

- `std::unreachable()` is in `<utility>` since C++23.
- MSVC equivalent: `__assume(false)`; GCC/Clang: `__builtin_unreachable()`.
- Use only when you can **prove** the path is impossible (closed enums, validated inputs).
- Combine with debug asserts: assert in debug, unreachable in release.
- If unsure, prefer `assert(false)` — it's safer, even without optimization benefits.
