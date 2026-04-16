# Understand ABI stability and binary compatibility concerns

**Category:** Best Practices & Idioms  
**Item:** #142  
**Reference:** <https://en.cppreference.com/w/cpp/language/abi_tag>  

---

## Topic Overview

**ABI (Application Binary Interface)** defines how compiled code interacts at the binary level: object layout, name mangling, calling conventions, vtable layout. Changing ABI means recompiling all dependent code.

### ABI-Breaking vs ABI-Safe Changes

| Change | ABI breaks? | Why |
| --- | --- | --- |
| Add data member | ✅ YES | Changes `sizeof(T)` |
| Reorder data members | ✅ YES | Changes member offsets |
| Add virtual function | ✅ YES | Changes vtable layout |
| Reorder virtual functions | ✅ YES | Changes vtable indices |
| Change function signature | ✅ YES | Changes mangled name |
| Add non-virtual member function | ❌ No | Not in class layout |
| Add static member function | ❌ No | No instance layout change |
| Add enum value at end | ❌ No | Enum value, not object layout |
| Change function body (implementation) | ❌ No | Layout unchanged |

---

## Self-Assessment

### Q1: Explain what changes break ABI

```cpp

#include <iostream>

// Version 1.0 of your library:
class Widget_v1 {
    int x_;          // offset 0
    int y_;          // offset 4
public:
    virtual void draw();  // vtable slot 0
    int x() const { return x_; }
    int y() const { return y_; }
};
// sizeof(Widget_v1) = 16 (vtable ptr + 2 ints, with padding)

// Version 2.0 — ABI-BREAKING changes:
class Widget_v2 {
    int x_;          // offset 0
    double z_;       // ADDED: shifts y_ offset!
    int y_;          // was at offset 4, now at offset 12
public:
    virtual void draw();    // vtable slot 0
    virtual void resize();  // ADDED: changes vtable layout!
    int x() const { return x_; }
    int y() const { return y_; }
};
// sizeof(Widget_v2) = 32 (different!)

// Code compiled against v1 that calls widget->y() will read
// from offset 4, but in v2 that's now part of z_.
// Result: WRONG VALUE or CRASH!

int main() {
    std::cout << "v1 size: " << sizeof(Widget_v1) << '\n';
    std::cout << "v2 size: " << sizeof(Widget_v2) << '\n';
    std::cout << "Sizes differ = ABI break!\n";
}
// Expected output:
// v1 size: 16
// v2 size: 32
// Sizes differ = ABI break!

```

### Q2: Show how Pimpl maintains ABI stability across library versions

```cpp

#include <iostream>
#include <memory>

// PUBLIC HEADER (stable ABI):
class Widget {
public:
    Widget();
    ~Widget();
    void draw() const;
    int value() const;

private:
    struct Impl;                    // forward declaration only
    std::unique_ptr<Impl> pimpl_;   // always sizeof(unique_ptr) = 8
};
// sizeof(Widget) NEVER CHANGES regardless of what's in Impl

// IMPLEMENTATION FILE (can change freely):
struct Widget::Impl {
    int x_ = 10;
    int y_ = 20;
    // VERSION 2: can add fields without breaking ABI!
    // double z_ = 30;
    // std::string name_ = "widget";
};

Widget::Widget() : pimpl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;

void Widget::draw() const {
    std::cout << "Drawing at (" << pimpl_->x_ << ", " << pimpl_->y_ << ")\n";
}

int Widget::value() const {
    return pimpl_->x_ + pimpl_->y_;
}

int main() {
    Widget w;
    w.draw();
    std::cout << "Value: " << w.value() << '\n';
    std::cout << "sizeof(Widget): " << sizeof(Widget) << '\n';
    // sizeof stays the same even if Impl changes!
}
// Expected output:
// Drawing at (10, 20)
// Value: 30
// sizeof(Widget): 8

```

### Q3: Describe the difference between API compatibility and ABI compatibility

**API (Application Programming Interface) compatibility:**

- Source-level: can you compile code written against the old version with the new version?
- Checked by the compiler.

**ABI (Application Binary Interface) compatibility:**

- Binary-level: can you link/load code compiled against the old version with the new library binary?
- NOT checked by any tool at link time — mismatches cause silent corruption.

```cpp

API compatible, ABI compatible:
  Old code compiles AND runs with new library binary.
  Example: adding a new free function to the library.

API compatible, ABI INCOMPATIBLE:
  Old code compiles with new headers but crashes with new binary.
  Example: adding a data member (old binary expects old sizeof).

API INCOMPATIBLE, ABI INCOMPATIBLE:
  Old code doesn't compile AND doesn't run.
  Example: renaming a function.

API INCOMPATIBLE, ABI compatible:
  Old code doesn't compile but old binary still works.
  Example: making a function [[nodiscard]] (source warning, same binary).

```

**Comparison:**

| Aspect | API compatibility | ABI compatibility |
| --- | --- | --- |
| Level | Source code | Binary (object code) |
| Detected by | Compiler | Nothing (crashes at runtime) |
| Example break | Rename function | Add data member |
| Fix | Recompile | Recompile ALL dependents |
| Maintained by | Header design | Object layout stability |

---

## Notes

- The C++ standard does not define ABI. Each platform has its own (Itanium C++ ABI on Linux, MSVC ABI on Windows).
- libstdc++ and libc++ have different ABIs — you can't mix them.
- Techniques for ABI stability: Pimpl, abstract interfaces (pure virtual), C-style `extern "C"` functions.
- `inline namespace` can be used for ABI versioning (e.g., `namespace v2 { ... }`).
- COM (Windows) and D-Bus (Linux) are ABI-stable interface technologies.
