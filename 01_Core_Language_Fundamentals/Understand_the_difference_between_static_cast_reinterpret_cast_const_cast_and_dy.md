# Understand the difference between static_cast, reinterpret_cast, const_cast, and dynamic_cast

**Category:** Core Language Fundamentals  
**Item:** #303  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/cast_operator>  

---

## Topic Overview

C++ provides four named casts, each serving a distinct purpose. Using the right cast expresses intent and enables compiler checks — unlike C-style casts which bypass everything.

### The Four Named Casts

| Cast | Purpose | Runtime cost | Safety |
| --- | --- | --- | --- |
| `static_cast` | Well-defined conversions (numeric, up/down hierarchy) | None | Medium — UB if wrong downcast |
| `dynamic_cast` | Safe polymorphic downcast (checks at runtime) | RTTI lookup | High — returns null/throws on failure |
| `const_cast` | Add/remove `const`/`volatile` | None | Low — UB if object is truly const |
| `reinterpret_cast` | Bitwise reinterpretation of pointer/reference | None | Very low — almost always UB |

### static_cast

```cpp

// Numeric conversion
double d = 3.14;
int i = static_cast<int>(d);  // 3 — truncation, well-defined

// Upcast (always safe)
struct Base {};
struct Derived : Base {};
Derived d;
Base* bp = static_cast<Base*>(&d);  // OK — always valid

// Downcast (UNSAFE if wrong type!)
Base* bp2 = new Derived;
Derived* dp = static_cast<Derived*>(bp2);  // OK here — really is Derived
// But if bp2 pointed to Base, this would be UB

```

### dynamic_cast

```cpp

struct Base { virtual ~Base() = default; };  // Must have virtual function!
struct Derived : Base { void special() {} };
struct Other : Base {};

Base* bp = new Other;
Derived* dp = dynamic_cast<Derived*>(bp);  // Returns nullptr — not a Derived
if (dp) { dp->special(); }                 // Safe — checked

// With references: throws std::bad_cast on failure
try {
    Derived& dr = dynamic_cast<Derived&>(*bp);
} catch (std::bad_cast& e) {
    std::cout << "Cast failed: " << e.what() << "\n";
}

```

### const_cast

```cpp

void legacy_api(char* p);  // Can't change this C API

const char* msg = "hello";
legacy_api(const_cast<char*>(msg));  // Remove const for C API
// DANGER: if legacy_api modifies the string → UB (string literals are const)

// Safe use: when you know the object is non-const
int x = 42;
const int& cr = x;
int& r = const_cast<int&>(cr);  // OK — x is not actually const
r = 100;  // Well-defined

```

### reinterpret_cast

```cpp

int x = 42;
int* ip = &x;

// Pointer to integer and back
uintptr_t addr = reinterpret_cast<uintptr_t>(ip);
int* ip2 = reinterpret_cast<int*>(addr);
// OK — round-tripping pointer through uintptr_t is well-defined

// Type punning — USUALLY UB (violates strict aliasing)
float f = 3.14f;
int bits = reinterpret_cast<int&>(f);  // UB! Use std::bit_cast instead

```

---

## Self-Assessment

### Q1: List one safe and one unsafe use case for each of the four named casts

**Answer:**

| Cast | Safe Use | Unsafe Use |
| --- | --- | --- |
| `static_cast` | `int` to `double` conversion | Downcast to wrong derived type → UB |
| `dynamic_cast` | Checking if `Base*` is really `Derived*` | N/A (always safe — returns null on failure) |
| `const_cast` | Passing to C API that doesn't modify | Removing `const` from a string literal then writing → UB |
| `reinterpret_cast` | Round-trip `ptr → uintptr_t → ptr` | Treating `float*` as `int*` (strict aliasing violation) |

```cpp

#include <iostream>
#include <cstdint>

struct Base { virtual ~Base() = default; };
struct A : Base { int a = 1; };
struct B : Base { int b = 2; };

int main() {
    // static_cast safe: numeric
    double pi = 3.14159;
    int truncated = static_cast<int>(pi);  // 3 — well-defined

    // static_cast UNSAFE: wrong downcast
    Base* bp = new B;
    // A* ap = static_cast<A*>(bp);  // UB! bp points to B, not A

    // dynamic_cast safe: runtime check
    A* ap2 = dynamic_cast<A*>(bp);  // nullptr — safe failure
    std::cout << "dynamic_cast to A: " << (ap2 ? "success" : "null") << "\n";

    // const_cast safe: non-const object
    int x = 42;
    const int& cr = x;
    const_cast<int&>(cr) = 100;  // OK — x is not const
    std::cout << "x = " << x << "\n";  // 100

    // reinterpret_cast safe: pointer round-trip
    int* ptr = &x;
    auto addr = reinterpret_cast<uintptr_t>(ptr);
    int* ptr2 = reinterpret_cast<int*>(addr);
    std::cout << "*ptr2 = " << *ptr2 << "\n";  // 100

    delete bp;
}

```

### Q2: Show that reinterpret_cast may generate a different address for pointers to the same object

```cpp

#include <iostream>

struct A { int a; };
struct B { double b; };
struct C : A, B { int c; };

int main() {
    C obj;
    obj.a = 1;
    obj.b = 2.0;
    obj.c = 3;

    C* cp = &obj;
    B* bp_static = static_cast<B*>(cp);         // Correct: adjusts pointer
    B* bp_reinterpret = reinterpret_cast<B*>(cp); // WRONG: no adjustment!

    std::cout << "C* address:              " << cp << "\n";
    std::cout << "static_cast<B*>:         " << bp_static << "\n";
    std::cout << "reinterpret_cast<B*>:    " << bp_reinterpret << "\n";

    // static_cast<B*> gives the CORRECT address of the B subobject
    // reinterpret_cast<B*> gives the SAME address as C* — WRONG!
    // This is because C layout is: [A subobject][B subobject][c member]
    // B subobject starts at an offset, not at the beginning

    std::cout << "bp_static->b:         " << bp_static->b << "\n";        // 2.0 ✓
    // std::cout << bp_reinterpret->b;  // UB! Reading A's memory as double
}

```

**How this works:**

- With multiple inheritance, `static_cast` **adjusts** the pointer to the correct subobject address.
- `reinterpret_cast` performs **no adjustment** — it keeps the same bit pattern.
- This means `reinterpret_cast<B*>(cp)` may point to the wrong subobject, causing UB.

### Q3: Demonstrate dynamic_cast failing and returning nullptr for an invalid downcast

```cpp

#include <iostream>

struct Animal {
    virtual ~Animal() = default;
    virtual void speak() { std::cout << "...\n"; }
};

struct Dog : Animal {
    void speak() override { std::cout << "Woof!\n"; }
    void fetch() { std::cout << "Fetching ball!\n"; }
};

struct Cat : Animal {
    void speak() override { std::cout << "Meow!\n"; }
    void purr() { std::cout << "Purrrr...\n"; }
};

void try_make_it_fetch(Animal* a) {
    // dynamic_cast checks at runtime if 'a' really points to a Dog
    Dog* dog = dynamic_cast<Dog*>(a);
    if (dog) {
        dog->fetch();  // Safe — confirmed it's a Dog
    } else {
        std::cout << "Not a dog — can't fetch!\n";
    }
}

int main() {
    Dog rex;
    Cat whiskers;

    Animal* a1 = &rex;
    Animal* a2 = &whiskers;

    try_make_it_fetch(a1);  // "Fetching ball!" — rex IS a Dog
    try_make_it_fetch(a2);  // "Not a dog — can't fetch!" — whiskers is a Cat

    // With references: bad_cast exception
    try {
        Dog& d = dynamic_cast<Dog&>(*a2);  // throws!
    } catch (const std::bad_cast& e) {
        std::cout << "bad_cast: " << e.what() << "\n";
    }

    // dynamic_cast with nullptr
    Animal* null_ptr = nullptr;
    Dog* d2 = dynamic_cast<Dog*>(null_ptr);
    std::cout << "null cast: " << (d2 == nullptr ? "nullptr" : "error") << "\n";
    // Always returns nullptr for nullptr input
}

```

**How this works:**

- `dynamic_cast` uses RTTI (Run-Time Type Information) to check the actual type at runtime.
- For pointers: returns `nullptr` on failure.
- For references: throws `std::bad_cast` on failure (since references can't be null).
- Requires at least one `virtual` function in the base class (to enable RTTI).
- Has a small runtime cost — avoid in tight loops; prefer `static_cast` when the type is known.

---

## Notes

- **Prefer `static_cast`** for known-safe conversions (numeric, upcast, enum ↔ int).
- **Use `dynamic_cast`** only when you genuinely don't know the runtime type — it's expensive.
- **Avoid `reinterpret_cast`** except for `uintptr_t` round-trips and interfacing with hardware/C APIs.
- **Minimize `const_cast`** — it usually indicates a design problem. Wrapping C APIs is the main valid use.
- C++20 `std::bit_cast` replaces many `reinterpret_cast` uses for type punning (well-defined for trivially copyable types).
- C-style casts `(Type)expr` try casts in order: `const_cast`, `static_cast`, `reinterpret_cast` — avoid them because they hide which conversion happens.
