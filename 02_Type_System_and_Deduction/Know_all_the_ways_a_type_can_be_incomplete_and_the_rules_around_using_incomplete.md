# Know all the ways a type can be incomplete and the rules around using incomplete types

**Category:** Type System & Deduction  
**Item:** #297  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/type#Incomplete_type>  

---

## Topic Overview

An **incomplete type** is a type that has been declared but not yet defined. The compiler knows the name exists but doesn't know the type's size or layout.

### Ways a Type Can Be Incomplete

```cpp

// 1. Forward-declared class/struct
class Widget;               // Incomplete — no definition yet
struct Node;                // Incomplete

// 2. Forward-declared enum (C++11, with underlying type)
enum class Color : int;     // Incomplete — enumerators unknown

// 3. void — always incomplete
// void x;                  // ❌ ERROR: cannot create object of type void

// 4. Arrays of unknown bound
extern int arr[];           // Incomplete — size unknown

// 5. Class being defined (inside its own body)
struct Self {
    // 'Self' is incomplete here — can't have Self member
    // Self s;              // ❌ ERROR: incomplete type
    Self* next;             // ✅ OK: pointer to incomplete type
};

```

### What You CAN Do With Incomplete Types

```cpp

class Widget;  // Forward declaration only

Widget* ptr;                    // ✅ Declare pointer
Widget& ref = *ptr;             // ✅ Declare reference
Widget* make_widget();          // ✅ Return type in function declaration
void process(Widget* w);        // ✅ Parameter type (pointer/reference)
extern Widget global_widget;    // ✅ extern declaration
using WidgetPtr = Widget*;      // ✅ Type alias involving pointer
std::unique_ptr<Widget> up;     // ✅ unique_ptr can hold incomplete type

```

### What You CANNOT Do With Incomplete Types

```cpp

class Widget;  // Forward declaration only

// Widget w;                   // ❌ Create instance (need size)
// sizeof(Widget);             // ❌ Get size
// Widget arr[10];             // ❌ Create array
// std::vector<Widget> v;      // ❌ Most containers need complete type
// ptr->method();              // ❌ Access members (layout unknown)
// delete ptr;                 // ⚠ UB without complete type (no dtor call)
// dynamic_cast<Derived*>(ptr) // ❌ Needs complete type info

```

### std::unique_ptr and Incomplete Types

`std::unique_ptr<T>` only requires `T` to be complete at the **point of destruction** (where the deleter is called):

```cpp

// widget.h
class Widget;  // Forward declaration
class Container {
    std::unique_ptr<Widget> pImpl;  // ✅ OK: Widget is incomplete here
public:
    Container();
    ~Container();  // Declared but NOT defined in header
};

// widget.cpp
#include "widget.h"
class Widget { /* full definition */ };
Container::Container() = default;
Container::~Container() = default;  // Widget is complete here → deleter works

```

---

## Self-Assessment

### Q1: Show that forward-declared types can be used in pointers/references but not instantiated

```cpp

#include <iostream>
#include <memory>

// Forward declaration — Widget is INCOMPLETE from this point
class Widget;

// ✅ Functions with pointers/references to incomplete types
Widget* create_widget();
void destroy_widget(Widget* w);
void process(Widget& w);

// ✅ Class using pointer to incomplete type
class Manager {
    Widget* widget_ = nullptr;        // ✅ Pointer OK
    // Widget widget_;                 // ❌ ERROR: incomplete type
    // std::vector<Widget> widgets_;   // ❌ ERROR: needs complete type
public:
    void init();
    void cleanup();
};

// --- Now define Widget (becomes complete) ---
class Widget {
    int id_;
    std::string name_;
public:
    Widget(int id, std::string name) : id_(id), name_(std::move(name)) {}
    void print() const { std::cout << "Widget(" << id_ << ", " << name_ << ")\n"; }
    int id() const { return id_; }
};

// All operations now work because Widget is complete:
Widget* create_widget() { return new Widget(1, "Test"); }
void destroy_widget(Widget* w) { delete w; }
void process(Widget& w) { w.print(); }

void Manager::init() { widget_ = create_widget(); }
void Manager::cleanup() { destroy_widget(widget_); widget_ = nullptr; }

int main() {
    Widget w(42, "Direct");  // ✅ Now complete — can instantiate
    w.print();               // ✅ Can access members

    Widget* ptr = create_widget();
    process(*ptr);           // ✅ Dereference and use
    destroy_widget(ptr);

    Manager m;
    m.init();
    m.cleanup();
}

```

**How this works:**

- Forward declaration gives the compiler just the name — enough for pointers and references (fixed size: 8 bytes on 64-bit).
- The compiler needs the full definition to know the size (for stack allocation), the layout (for member access), and the destructor (for deletion).

### Q2: Explain why `std::unique_ptr<T>` can work with an incomplete `T`

**Answer:**

`unique_ptr<T>` stores a pointer (`T*`) — which has a known size regardless of whether `T` is complete. The critical operation that needs `T` to be complete is **deletion** (calling the destructor), which only happens inside `unique_ptr`'s destructor.

```cpp

#include <iostream>
#include <memory>

// header.h — Widget is incomplete
class Widget;

class Pimpl {
    std::unique_ptr<Widget> impl_;  // ✅ OK: only stores T*
public:
    Pimpl();
    ~Pimpl();                        // Declare — do NOT define in header!
    // If we let the compiler generate ~Pimpl() here, it would instantiate
    // unique_ptr<Widget>::~unique_ptr(), which calls delete on Widget.
    // But Widget is incomplete here → UB (no destructor call) or compiler error.

    Pimpl(Pimpl&&);                  // Same issue — must declare, define in .cpp
    Pimpl& operator=(Pimpl&&);

    void use();
};

// source.cpp — Widget is complete
class Widget {
public:
    int value = 42;
    ~Widget() { std::cout << "Widget destroyed\n"; }
};

Pimpl::Pimpl() : impl_(std::make_unique<Widget>()) {}
Pimpl::~Pimpl() = default;          // ✅ Widget is complete here
Pimpl::Pimpl(Pimpl&&) = default;
Pimpl& Pimpl::operator=(Pimpl&&) = default;

void Pimpl::use() {
    std::cout << "Widget value: " << impl_->value << "\n";
}

int main() {
    Pimpl p;
    p.use();
}  // "Widget destroyed" — destructor properly called

```

**Key rule:** `std::unique_ptr<T>` declaration = incomplete OK. `std::unique_ptr<T>` destruction = needs complete `T`.

**Contrast with `std::shared_ptr<T>`:** `shared_ptr` captures the deleter at *construction* time (type-erased), so even `~shared_ptr()` doesn't need `T` to be complete. However, the *constructor* does need complete `T`.

### Q3: Use incomplete types to break circular header dependencies

```cpp

#include <iostream>
#include <memory>
#include <string>

// Problem: Engine needs Window*, Window needs Engine*
// If both headers #include each other → circular dependency!

// ---- engine.h ----
class Window;  // Forward declaration breaks the cycle!

class Engine {
    Window* main_window_ = nullptr;  // ✅ Pointer to incomplete type
public:
    void set_window(Window* w) { main_window_ = w; }
    void render();  // Defined in .cpp where Window is complete
};

// ---- window.h ----
class Engine;  // Forward declaration

class Window {
    Engine* engine_ = nullptr;       // ✅ Pointer to incomplete type
    std::string title_;
public:
    Window(std::string title) : title_(std::move(title)) {}
    void set_engine(Engine* e) { engine_ = e; }
    void show();    // Defined in .cpp where Engine is complete
    const std::string& title() const { return title_; }
};

// ---- engine.cpp ----
// #include "engine.h"
// #include "window.h"  // Full definition now available
void Engine::render() {
    if (main_window_) {
        std::cout << "Rendering in window: " << main_window_->title() << "\n";
    }
}

// ---- window.cpp ----
// #include "window.h"
// #include "engine.h"  // Full definition now available
void Window::show() {
    std::cout << "Showing window: " << title_ << "\n";
    if (engine_) {
        engine_->render();
    }
}

int main() {
    Engine engine;
    Window window("Main Window");

    engine.set_window(&window);
    window.set_engine(&engine);

    window.show();
    // Output:
    // Showing window: Main Window
    // Rendering in window: Main Window
}

```

**Pattern:**

1. In header: use forward declarations for types only used via pointer/reference.
2. In `.cpp` file: `#include` the full definition header — now the type is complete.
3. This eliminates circular `#include` chains that cause compilation errors.

---

## Notes

- Forward declare classes with `class Foo;` — it works for both `class` and `struct` (the keyword doesn't need to match the definition).
- `std::shared_ptr<T>` can also hold incomplete types but needs `T` complete at the point of construction.
- The Pimpl idiom (Pointer to IMPLementation) is the canonical use of incomplete types — it hides implementation details and reduces compile-time dependencies.
- `static_assert(sizeof(T) > 0)` or `static_assert(!std::is_void_v<T>)` can be used to catch accidental use of incomplete types.
- You cannot inherit from an incomplete type, `static_cast` through an incomplete hierarchy, or use `typeid` on an incomplete type.

```cpp

// Your practice code

```
