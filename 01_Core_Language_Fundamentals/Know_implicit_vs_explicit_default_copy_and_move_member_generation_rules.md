# Know implicit vs explicit default, copy, and move member generation rules

**Category:** Core Language Fundamentals  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/language/default_constructor>  

---

## Topic Overview

### The Six Special Member Functions

Every class potentially has these six special member functions:

| # | Function | Signature |
| --- | --- | --- |
| 1 | Default constructor | `T()` |
| 2 | Destructor | `~T()` |
| 3 | Copy constructor | `T(const T&)` |
| 4 | Copy assignment | `T& operator=(const T&)` |
| 5 | Move constructor | `T(T&&)` |
| 6 | Move assignment | `T& operator=(T&&)` |

The compiler can **implicitly generate** any of these, but the rules for when it does are complex and critical to understand.

### The Generation Rules Table (The "Rule of Zero/Three/Five" Foundation)

| You declare... | Default ctor | Destructor | Copy ctor | Copy assign | Move ctor | Move assign |
| --- | --- | --- | --- | --- | --- | --- |
| Nothing | ✅ implicit | ✅ implicit | ✅ implicit | ✅ implicit | ✅ implicit | ✅ implicit |
| Any constructor | ❌ suppressed | ✅ implicit | ✅ implicit | ✅ implicit | ✅ implicit | ✅ implicit |
| Destructor | ✅ implicit | user | ✅ implicit* | ✅ implicit* | ❌ suppressed | ❌ suppressed |
| Copy constructor | ❌ suppressed | ✅ implicit | user | ✅ implicit* | ❌ suppressed | ❌ suppressed |
| Copy assignment | ✅ implicit | ✅ implicit | ✅ implicit* | user | ❌ suppressed | ❌ suppressed |
| Move constructor | ❌ suppressed | ✅ implicit | =delete | =delete | user | ❌ suppressed |
| Move assignment | ✅ implicit | ✅ implicit | ✅ implicit* | ✅ implicit* | ❌ suppressed | user |

`*` = deprecated implicit generation (C++11) — works but the committee considers it a defect. Use `= default` explicitly.

### Key Rules

1. **Declaring ANY constructor** (including parameterized) suppresses the implicit default constructor.
2. **Declaring a destructor** suppresses implicit move operations.
3. **Declaring a copy operation** suppresses implicit move operations.
4. **Declaring a move operation** suppresses implicit copy operations (they become `= delete`).
5. **`= default`** explicitly requests the compiler-generated version.
6. **`= delete`** explicitly disables a function.

### The Rule of Zero

If your class doesn't directly manage resources, **don't declare any special members** — let the compiler generate them all:

```cpp

struct Person {
    std::string name;
    int age;
    // Rule of Zero: compiler generates all 6 correctly
    // because std::string and int handle themselves
};

```

### The Rule of Five

If you declare **any** of destructor/copy constructor/copy assignment, declare **all five**:

```cpp

class Buffer {
    int* data_;
    std::size_t size_;
public:
    Buffer(std::size_t n) : data_(new int[n]), size_(n) {}

    ~Buffer()                          { delete[] data_; }
    Buffer(const Buffer& o)            : data_(new int[o.size_]), size_(o.size_) {
        std::copy(o.data_, o.data_ + size_, data_);
    }
    Buffer& operator=(const Buffer& o) {
        if (this != &o) { Buffer tmp(o); swap(*this, tmp); }
        return *this;
    }
    Buffer(Buffer&& o) noexcept        : data_(o.data_), size_(o.size_) {
        o.data_ = nullptr; o.size_ = 0;
    }
    Buffer& operator=(Buffer&& o) noexcept {
        if (this != &o) { delete[] data_; data_ = o.data_; size_ = o.size_;
            o.data_ = nullptr; o.size_ = 0; }
        return *this;
    }

    friend void swap(Buffer& a, Buffer& b) noexcept {
        std::swap(a.data_, b.data_);
        std::swap(a.size_, b.size_);
    }
};

```

---

## Self-Assessment

### Q1: Show that declaring any constructor suppresses the implicit default constructor

```cpp

#include <iostream>
#include <string>

struct Widget {
    int value;
    std::string name;

    // Declaring this parameterized constructor suppresses the default constructor
    Widget(int v) : value(v), name("unnamed") {
        std::cout << "Parameterized ctor: " << value << "\n";
    }
};

struct Gadget {
    int value;
    std::string name;
    // No constructors declared → default constructor is implicitly generated
};

struct Restored {
    int value;
    std::string name;

    Restored(int v) : value(v), name("unnamed") {}

    // Explicitly bring back the default constructor
    Restored() = default;  // <-- restores it!
};

int main() {
    // Gadget g;      // OK: implicit default constructor
    // Widget w;      // ERROR: no default constructor!
    //                // error: no matching function for call to 'Widget::Widget()'

    Widget w(42);      // OK: uses the declared constructor
    std::cout << "w = " << w.value << "\n";

    Restored r;        // OK: = default brings it back
    std::cout << "r.value = " << r.value << "\n"; // indeterminate (default-initialized)

    Restored r2(10);   // Also OK
    std::cout << "r2.value = " << r2.value << "\n"; // 10

    return 0;
}

```

**How this works:**

- `Widget` declares `Widget(int)` → the compiler no longer generates `Widget()`.
- Attempting `Widget w;` produces a compile error.
- `Restored` declares `Restored(int)` too, but explicitly adds `Restored() = default;` to get the default constructor back.
- This rule applies regardless of constructor type: copy constructor, move constructor, conversion constructor — any user-declared constructor suppresses the default constructor.

### Q2: Demonstrate that declaring a destructor suppresses implicit move constructor generation

```cpp

#include <iostream>
#include <string>
#include <type_traits>

struct NoDestructor {
    std::string data;
    // No user-declared destructor → move operations are implicitly generated
};

struct WithDestructor {
    std::string data;
    ~WithDestructor() { /* user-declared destructor */ }
    // Move constructor: NOT generated (suppressed)
    // Move assignment: NOT generated (suppressed)
    // Copy constructor: still generated (but deprecated!)
    // Copy assignment: still generated (but deprecated!)
};

struct Fixed {
    std::string data;
    ~Fixed() = default;
    // Even "= default" counts as user-declared!
    // Move operations are still suppressed!
    // But since it's trivial, some compilers may optimize anyway
};

struct ProperlyFixed {
    std::string data;
    ~ProperlyFixed() = default;
    ProperlyFixed() = default;
    ProperlyFixed(const ProperlyFixed&) = default;
    ProperlyFixed& operator=(const ProperlyFixed&) = default;
    ProperlyFixed(ProperlyFixed&&) = default;            // Explicitly requested
    ProperlyFixed& operator=(ProperlyFixed&&) = default; // Explicitly requested
};

int main() {
    // Check move constructibility:
    std::cout << "NoDestructor move constructible: "
              << std::is_move_constructible_v<NoDestructor> << "\n";     // 1
    std::cout << "WithDestructor move constructible: "
              << std::is_move_constructible_v<WithDestructor> << "\n";   // 1 (falls back to copy!)
    std::cout << "WithDestructor nothrow move constructible: "
              << std::is_nothrow_move_constructible_v<WithDestructor> << "\n"; // 0! (copy might throw)

    // The difference: WithDestructor "moves" by copying (expensive!)
    NoDestructor nd;
    nd.data = std::string(1000, 'x');
    NoDestructor nd2 = std::move(nd);
    std::cout << "After move, nd.data.size() = " << nd.data.size() << "\n";  // 0 (moved from)

    WithDestructor wd;
    wd.data = std::string(1000, 'x');
    WithDestructor wd2 = std::move(wd);
    std::cout << "After 'move', wd.data.size() = " << wd.data.size() << "\n"; // 1000! (copied, not moved!)

    return 0;
}

```

**Output:**

```text

NoDestructor move constructible: 1
WithDestructor move constructible: 1
WithDestructor nothrow move constructible: 0
After move, nd.data.size() = 0
After 'move', wd.data.size() = 1000

```

**How this works:**

- `NoDestructor` has all special members implicitly generated → `std::move` triggers the move constructor → string is moved (source becomes empty).
- `WithDestructor` has a user-declared destructor → move constructor is NOT generated → `std::move(wd)` casts to rvalue, but overload resolution picks the **copy constructor** (which accepts `const T&`) → string is COPIED, not moved.
- This is a **silent performance bug** — the code compiles and runs correctly, but moves are silently degraded to copies.
- `= default` on the destructor still counts as user-declared — it still suppresses moves!

### Q3: Use `= default` and `= delete` to control generation

```cpp

#include <iostream>
#include <string>

// Example 1: Non-copyable, movable resource handle
class FileHandle {
    int fd_;
public:
    explicit FileHandle(int fd) : fd_(fd) {}
    ~FileHandle() { if (fd_ >= 0) { /* close(fd_); */ } }

    // Delete copy operations — file handles shouldn't be duplicated
    FileHandle(const FileHandle&) = delete;
    FileHandle& operator=(const FileHandle&) = delete;

    // Explicitly default move operations
    FileHandle(FileHandle&& other) noexcept : fd_(other.fd_) { other.fd_ = -1; }
    FileHandle& operator=(FileHandle&& other) noexcept {
        if (this != &other) { fd_ = other.fd_; other.fd_ = -1; }
        return *this;
    }
};

// Example 2: Copyable, non-movable (rare but valid)
class SharedState {
    std::string data_;
public:
    SharedState(std::string d) : data_(std::move(d)) {}
    SharedState(const SharedState&) = default;
    SharedState& operator=(const SharedState&) = default;
    SharedState(SharedState&&) = delete;            // Explicitly non-movable
    SharedState& operator=(SharedState&&) = delete;
};

// Example 3: Neither copyable nor movable (singleton pattern)
class Singleton {
public:
    static Singleton& instance() {
        static Singleton s;
        return s;
    }

    Singleton(const Singleton&) = delete;
    Singleton& operator=(const Singleton&) = delete;
    Singleton(Singleton&&) = delete;
    Singleton& operator=(Singleton&&) = delete;

private:
    Singleton() = default;
    ~Singleton() = default;
};

// Resulting special member table:
//
// | Class       | Default | Dtor    | Copy ctor | Copy =  | Move ctor | Move =  |
// |-------------|---------|---------|-----------|---------|-----------|---------|
// | FileHandle  | ❌      | user    | =delete   | =delete | user      | user    |
// | SharedState | ❌      | default | =default  | =default| =delete   | =delete |
// | Singleton   | =default| =default| =delete   | =delete | =delete   | =delete |

int main() {
    FileHandle fh(42);
    // FileHandle fh2 = fh;              // ERROR: copy deleted
    FileHandle fh2 = std::move(fh);      // OK: move is defined

    SharedState ss("hello");
    SharedState ss2 = ss;                 // OK: copy works
    // SharedState ss3 = std::move(ss);   // ERROR: move deleted

    Singleton& s = Singleton::instance();
    // Singleton s2 = s;                  // ERROR: copy deleted
    // Singleton s3 = std::move(s);       // ERROR: move deleted

    return 0;
}

```

**How this works:**

- `= default` tells the compiler: "Generate the default version of this function."
- `= delete` tells the compiler: "This function exists in overload resolution but cannot be called."
- Deleted functions participate in overload resolution — calling one is a compile error (not SFINAE).
- The combination of `= default` and `= delete` gives you complete control over which operations are allowed.

---

## Additional Examples

### Detecting Suppressed Moves with Type Traits

```cpp

#include <type_traits>
#include <iostream>
#include <string>

struct A { std::string s; };                    // All defaults
struct B { std::string s; ~B() {} };            // Destructor suppresses moves
struct C { std::string s; C(const C&) = default; C& operator=(const C&) = default; }; // Copy suppresses moves
struct D { std::string s; ~D() = default;
           D() = default; D(const D&) = default; D& operator=(const D&) = default;
           D(D&&) = default; D& operator=(D&&) = default; }; // All explicit

int main() {
    std::cout << std::boolalpha;
    std::cout << "A nothrow_move: " << std::is_nothrow_move_constructible_v<A> << "\n"; // true
    std::cout << "B nothrow_move: " << std::is_nothrow_move_constructible_v<B> << "\n"; // false (copies!)
    std::cout << "C nothrow_move: " << std::is_nothrow_move_constructible_v<C> << "\n"; // false (copies!)
    std::cout << "D nothrow_move: " << std::is_nothrow_move_constructible_v<D> << "\n"; // true
}

```

---

## Notes

- **Rule of Zero:** If you don't manage resources directly, declare nothing — compiler does it right.
- **Rule of Five:** If you declare destructor, copy ctor, or copy assignment, declare all five.
- Declaring a destructor (even `= default`) suppresses move operations — this is the most common gotcha.
- Use `std::is_nothrow_move_constructible_v` to verify moves weren't silently replaced by copies.
- `= delete` participates in overload resolution — a deleted function is a better match than no function, producing a clear error.
