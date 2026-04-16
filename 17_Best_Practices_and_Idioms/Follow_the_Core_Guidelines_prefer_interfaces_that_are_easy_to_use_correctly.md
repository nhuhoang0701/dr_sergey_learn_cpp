# Follow the Core Guidelines: prefer interfaces that are easy to use correctly

**Category:** Best Practices & Idioms  
**Item:** #127  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>  

---

## Topic Overview

Scott Meyers: *"Make interfaces easy to use correctly and hard to use incorrectly."* The C++ Core Guidelines codify this with rules about parameter types, preconditions, and ownership.

### Common API Mistakes

| Anti-pattern | Better design |
| --- | --- |
| `void draw(int x, int y, int w, int h)` | `void draw(Rect r)` |
| `void set(bool visible, bool enabled)` | `void set(Visible v, Enabled e)` |
| `Widget* create()` — who deletes? | `unique_ptr<Widget> create()` |
| `void process(int* data, int size)` | `void process(span<int> data)` |

---

## Self-Assessment

### Q1: Redesign a function that takes four `int` parameters to use a named struct

```cpp

#include <iostream>

// BAD: What does each int mean? Easy to swap arguments by mistake.
void create_window_bad(int x, int y, int width, int height) {
    std::cout << "Window at (" << x << "," << y
              << ") size " << width << "x" << height << '\n';
}

// GOOD: Named struct with designated initializers (C++20)
struct WindowConfig {
    int x = 0;
    int y = 0;
    int width = 800;
    int height = 600;
};

void create_window(const WindowConfig& cfg) {
    std::cout << "Window at (" << cfg.x << "," << cfg.y
              << ") size " << cfg.width << "x" << cfg.height << '\n';
}

// ALSO GOOD: Strong types to prevent swapping
struct Position { int x, y; };
struct Size { int width, height; };

void create_window_typed(Position pos, Size size) {
    std::cout << "Window at (" << pos.x << "," << pos.y
              << ") size " << size.width << "x" << size.height << '\n';
}

int main() {
    // BAD: easy to swap width/height with x/y
    create_window_bad(100, 200, 800, 600);
    // create_window_bad(800, 600, 100, 200);  // oops! swapped pairs

    // GOOD: self-documenting with designated initializers
    create_window({.x = 100, .y = 200, .width = 800, .height = 600});

    // GOOD: strong types prevent misuse
    create_window_typed({100, 200}, {800, 600});
    // create_window_typed({800, 600}, {100, 200});  // still compiles but less likely
}
// Expected output:
// Window at (100,200) size 800x600
// Window at (100,200) size 800x600
// Window at (100,200) size 800x600

```

### Q2: Show how GSL enforces preconditions

```cpp

#include <iostream>
#include <stdexcept>
#include <cstdlib>
#include <span>

// GSL-style precondition macros (simplified)
#define Expects(cond) \
    do { if (!(cond)) { \
        std::cerr << "Precondition failed: " #cond \
                  << " at " << __FILE__ << ":" << __LINE__ << '\n'; \
        std::abort(); \
    }} while(0)

#define Ensures(cond) \
    do { if (!(cond)) { \
        std::cerr << "Postcondition failed: " #cond \
                  << " at " << __FILE__ << ":" << __LINE__ << '\n'; \
        std::abort(); \
    }} while(0)

// Function with preconditions
double safe_divide(double num, double den) {
    Expects(den != 0.0);  // precondition: no division by zero
    double result = num / den;
    Ensures(result == result);  // postcondition: not NaN
    return result;
}

// GSL's span (now std::span in C++20) prevents buffer overflows
void process(std::span<int> data) {
    Expects(!data.empty());  // precondition
    for (int& x : data)
        x *= 2;
}

int main() {
    std::cout << safe_divide(10.0, 3.0) << '\n';

    int arr[] = {1, 2, 3};
    process(arr);  // std::span deduces size
    for (int x : arr) std::cout << x << ' ';
    std::cout << '\n';
}
// Expected output:
// 3.33333
// 2 4 6

```

### Q3: Apply the Principle of Least Surprise to fix a confusing ownership model

```cpp

#include <iostream>
#include <memory>
#include <string>

// BAD: confusing ownership
class RegistryBad {
public:
    // Who owns the returned pointer? Caller? Registry?
    int* get_value(const std::string& key) {
        return &values_[0];  // raw pointer — ambiguous!
    }

    // Does the registry take ownership of p?
    void register_handler(void* p) {
        // Will it delete p? Who knows!
    }
private:
    int values_[10] = {};
};

// GOOD: ownership is clear from the API
class RegistryGood {
public:
    // Observing pointer: Registry owns it, caller just borrows
    const int* get_value(const std::string& key) const {
        return &values_[0];
    }

    // Transfer ownership: unique_ptr makes it explicit
    void register_handler(std::unique_ptr<int> p) {
        handler_ = std::move(p);  // registry now owns it
    }

    // Shared ownership: shared_ptr makes it explicit
    void register_shared(std::shared_ptr<int> p) {
        shared_ = std::move(p);  // both caller and registry may own it
    }

private:
    int values_[10] = {};
    std::unique_ptr<int> handler_;
    std::shared_ptr<int> shared_;
};

int main() {
    RegistryGood reg;

    // Ownership transfer is explicit
    reg.register_handler(std::make_unique<int>(42));

    // Shared ownership is explicit
    auto shared = std::make_shared<int>(100);
    reg.register_shared(shared);

    std::cout << "Shared use count: " << shared.use_count() << '\n';
}
// Expected output:
// Shared use count: 2

```

**Ownership guidelines summary:**

| Parameter type | Ownership meaning |
| --- | --- |
| `T*`, `T&` | Non-owning (borrowing) |
| `unique_ptr<T>` | Transferring ownership |
| `shared_ptr<T>` | Sharing ownership |
| `span<T>` | Non-owning view of contiguous data |
| `string_view` | Non-owning view of string |

---

## Notes

- C++ Core Guideline I.4: "Make interfaces precisely and strongly typed."
- C++ Core Guideline I.11: "Never transfer ownership by a raw pointer (T*) or reference (T&)."
- GSL is available at https://github.com/microsoft/GSL.
- `std::span` (C++20) replaces `gsl::span`; `std::string_view` (C++17) replaces `gsl::string_span`.
