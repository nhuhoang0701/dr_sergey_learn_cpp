# Know the exact rules for when a destructor is implicitly deleted

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/destructor>  

---

## Topic Overview

### When Is a Destructor Implicitly Deleted

The compiler generates an implicit destructor for any class that doesn't declare one. However, the implicit destructor is defined as **deleted** if any of these conditions hold. Think of it as the compiler saying "I can't write a safe destructor for you here":

1. **A non-static data member has a deleted or inaccessible destructor.**
2. **A direct or virtual base class has a deleted or inaccessible destructor.**
3. **The class has a variant member with a non-trivial destructor** (in a union-like class).
4. **The implicitly-defined destructor would invoke undefined behavior** (e.g., calling a pure virtual function).

When the destructor is deleted, you cannot:

- Create objects on the stack
- Use `delete` on heap-allocated objects
- Store them in containers
- Use them as data members (without custom handling)

### `unique_ptr` and Incomplete Types

The most common real-world encounter with deleted destructors involves `std::unique_ptr<IncompleteType>`. Here's the classic Pimpl trap:

```cpp
// header.h
class Impl;  // Forward declaration - incomplete type

class Widget {
    std::unique_ptr<Impl> pImpl;  // OK to declare
public:
    Widget();
    // If we don't declare ~Widget(), the compiler generates it HERE in the header.
    // But Impl is incomplete here, so unique_ptr<Impl>::~unique_ptr() can't call delete.
    // Result: the implicit destructor is DELETED (or a compile error).
};
```

The fix is to **declare the destructor in the header** and **define it in the .cpp** where `Impl` is complete:

```cpp
// header.h
class Widget {
    std::unique_ptr<Impl> pImpl;
public:
    Widget();
    ~Widget();  // Declared but not defined here
};

// source.cpp
#include "header.h"
class Impl { /* full definition */ };
Widget::Widget() : pImpl(std::make_unique<Impl>()) {}
Widget::~Widget() = default;  // Defined here - Impl is complete!
```

The out-of-line `= default` looks redundant, but it moves the code generation to the point where the type is complete.

---

## Self-Assessment

### Q1: List the conditions that cause the implicit destructor to be deleted

Each condition below maps to one of the four rules. They all compile as shown - the comments tell you what would fail and why:

```cpp
#include <iostream>
#include <memory>

// Condition 1: Member with deleted destructor
struct NoDestroy {
    ~NoDestroy() = delete;
};

struct A {
    NoDestroy nd;
    // ~A() is implicitly DELETED because NoDestroy::~NoDestroy() is deleted
};
// A a;  // ERROR: use of deleted function 'A::~A()'

// Condition 2: Base class with deleted/inaccessible destructor
struct PrivateDtor {
private:
    ~PrivateDtor() {}  // Inaccessible from derived classes
};

struct B : PrivateDtor {
    // ~B() is implicitly DELETED because PrivateDtor::~PrivateDtor() is inaccessible
};
// B b;  // ERROR

// Condition 3: Union with non-trivial destructor member
union U {
    std::string s;  // std::string has non-trivial destructor
    int i;
    // ~U() is implicitly DELETED - compiler doesn't know whether to destroy s or i
    // User must provide a destructor manually
};

// Condition 4: Virtual base with deleted destructor
struct VBase {
    ~VBase() = delete;
};
struct C : virtual VBase {
    // ~C() is implicitly DELETED
};

int main() {
    // Demonstrate the union fix:
    union SafeU {
        std::string s;
        int i;

        SafeU() : i(0) {}        // Must initialize one member
        ~SafeU() {}               // User-defined: knows which member to destroy
        // In practice, use std::variant instead of raw unions with non-trivial types
    };

    SafeU u;
    u.i = 42;  // OK

    std::cout << "sizeof(U): " << sizeof(U) << "\n";
    return 0;
}
```

If the table feels like a lot, the common thread is: the compiler can't delete something it doesn't know how to delete.

**Summary table:**

| Condition | Example | Fix |
| --- | --- | --- |
| Member with deleted dtor | `struct A { NoDestroy nd; };` | Remove the member or provide custom dtor |
| Base with inaccessible dtor | `struct B : PrivateDtor {};` | Make dtor protected or public |
| Union with non-trivial member | `union U { std::string s; };` | Use `std::variant` or manual dtor |
| `unique_ptr<IncompleteType>` | Pimpl pattern | Out-of-line destructor |

### Q2: Show that a class with `unique_ptr<IncompleteType>` needs an out-of-line destructor

This is one of the most confusing compiler errors you'll encounter in real codebases. The key insight is that `unique_ptr`'s destructor needs a complete type at the point where the destructor body is generated:

```cpp
// === widget.h ===
#include <memory>
#include <string>

class Impl;  // Forward declaration - incomplete type

class Widget {
    std::unique_ptr<Impl> pImpl_;
    std::string name_;
public:
    Widget(std::string name);

    // REQUIRED: declare destructor (but don't define here)
    // Without this, the compiler generates ~Widget() in every TU that includes this header
    // Since Impl is incomplete here, unique_ptr can't call delete - compilation error
    ~Widget();

    // Also need to declare move operations (destructor suppresses implicit moves)
    Widget(Widget&& other) noexcept;
    Widget& operator=(Widget&& other) noexcept;

    void do_work();
    std::string name() const;
};

// === widget.cpp ===
// #include "widget.h"
#include <iostream>

// Full definition of Impl - now the type is complete
class Impl {
public:
    int state = 0;
    void compute() { state++; std::cout << "Computing... state=" << state << "\n"; }
};

// Define all special members HERE where Impl is complete
Widget::Widget(std::string name)
    : pImpl_(std::make_unique<Impl>()), name_(std::move(name)) {}

Widget::~Widget() = default;  // OK - Impl is complete here

Widget::Widget(Widget&& other) noexcept = default;
Widget& Widget::operator=(Widget&& other) noexcept = default;

void Widget::do_work() { pImpl_->compute(); }
std::string Widget::name() const { return name_; }

// === main.cpp ===
int main() {
    Widget w("MyWidget");
    w.do_work();  // Computing... state=1
    w.do_work();  // Computing... state=2

    Widget w2 = std::move(w);  // Move works
    w2.do_work(); // Computing... state=3

    std::cout << "Name: " << w2.name() << "\n";
    return 0;
}
```

All the special member definitions live in the `.cpp` file, which is the one place that sees the full `Impl` definition.

Why this happens:

- `std::unique_ptr<T>::~unique_ptr()` calls `delete` on the stored `T*`.
- To call `delete`, the compiler must know the **complete type** of `T` (to call its destructor and compute size).
- In the header, `Impl` is only forward-declared (incomplete) - the compiler can't generate `~Widget()`.
- Defining `~Widget() = default` in the `.cpp` where `Impl` is fully defined resolves this.

### Q3: Explain why deleting the destructor prevents both stack and heap allocation

You might think "just don't call `delete`" - but the compiler needs confidence that it *can* call the destructor before it lets you create the object at all:

```cpp
#include <iostream>
#include <new>

struct NoDelete {
    int value;
    ~NoDelete() = delete;
};

int main() {
    // Stack allocation: FAILS
    // NoDelete x;  // ERROR: use of deleted function '~NoDelete()'
    // Compiler must call destructor when x goes out of scope - can't!

    // Heap allocation with delete: FAILS
    // NoDelete* p = new NoDelete{42};
    // delete p;  // ERROR: use of deleted function '~NoDelete()'

    // Even new[] fails:
    // NoDelete* arr = new NoDelete[5];
    // delete[] arr;  // ERROR

    // Placement new technically works (no automatic destruction needed):
    alignas(NoDelete) unsigned char buf[sizeof(NoDelete)];
    NoDelete* placed = new (buf) NoDelete{42};
    std::cout << placed->value << "\n";  // 42
    // But you can NEVER destroy it properly - memory leak on heap, UB on stack

    // Even in containers: FAILS
    // std::vector<NoDelete> v;  // ERROR: vector needs to destroy elements

    // WHY does regular new fail?
    // Because for regular `new NoDelete{42}`, the compiler generates code like:
    //   1. allocate memory
    //   2. construct object
    //   3. (if construction throws) call destructor - oops, deleted!
    // Even if the constructor succeeds, the compiler needs confidence that
    // the destructor is callable for exception safety.

    return 0;
}
```

Placement new into raw memory is the only escape - but then you can never properly clean up, so it's rarely what you want.

Why both stack AND heap are blocked:

- **Stack objects:** At the end of scope, the compiler inserts a call to `~T()`. If `~T()` is deleted, the code won't compile.
- **Heap objects (`new`/`delete`):** `delete p` calls `~T()` then frees memory. If `~T()` is deleted, `delete` is ill-formed. Even `new` may fail because the compiler generates exception cleanup code that calls the destructor.
- **The only escape:** Placement new into raw memory (no automatic cleanup), but then you can never properly destroy the object.

Practical use of `= delete` destructor:

- Prevent instantiation entirely (combine with private constructor for tag types)
- Some memory allocator designs use it to prevent accidental deallocation

---

## Notes

- A deleted destructor makes a type essentially unusable for normal object creation.
- The number one real-world case is `unique_ptr<IncompleteType>` in the Pimpl pattern - fix with out-of-line destructor.
- Unions with non-trivial members get a deleted destructor - prefer `std::variant`.
- When you define a destructor (even `= default`) for Pimpl, also define move operations explicitly.
- Use `std::is_destructible_v<T>` to check at compile time whether a type can be destroyed.
