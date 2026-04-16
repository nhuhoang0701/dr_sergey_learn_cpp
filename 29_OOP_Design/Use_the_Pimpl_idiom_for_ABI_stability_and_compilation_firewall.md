# Use the Pimpl idiom for ABI stability and compilation firewall

**Category:** OOP Design

---

## Topic Overview

The **Pointer-to-Implementation** (Pimpl) idiom hides private data and methods behind an opaque pointer, giving two key benefits:

| Benefit | How | Impact |
| --- | --- | --- |
| **Compilation firewall** | Private headers not included in public header | Faster incremental builds |
| **ABI stability** | Class size = 1 pointer, never changes | Library upgrades without recompile |
| **Reduced coupling** | Client code doesn't see private types | Cleaner dependency graph |

### Memory Layout

```cpp

Without Pimpl:               With Pimpl:
┌──────────────┐             ┌───────────┐      ┌──────────────┐
│ public data   │             │ Impl* ptr─┼─────►│ all private   │
│ private data  │             └───────────┘      │ data & deps   │
│ private deps  │             sizeof = 8 bytes   └──────────────┘
└──────────────┘                                 (heap allocated)
sizeof varies

```

---

## Self-Assessment

### Q1: Implement a proper Pimpl class with unique_ptr

**Answer:**

```cpp

// ─── widget.h (PUBLIC HEADER) ───
#pragma once
#include <memory>
#include <string>

class Widget {
public:
    explicit Widget(const std::string& name);
    ~Widget();  // Must declare in header!

    // Move-only (or implement copy via clone)
    Widget(Widget&&) noexcept;
    Widget& operator=(Widget&&) noexcept;

    void draw() const;
    void set_color(int r, int g, int b);

private:
    struct Impl;                  // Forward declaration only
    std::unique_ptr<Impl> pimpl_; // Opaque pointer
};

// ─── widget.cpp (PRIVATE IMPLEMENTATION) ───
#include "widget.h"
#include <iostream>
// Heavy headers only here — not exposed to clients
#include <vector>
#include <map>

struct Widget::Impl {
    std::string name;
    int r = 0, g = 0, b = 0;
    std::vector<float> vertices;  // Hidden from public header
    std::map<std::string, int> properties;

    void internal_render() const {
        std::cout << "Rendering " << name
                  << " color=(" << r << "," << g << "," << b << ")\n";
    }
};

// Definitions MUST be in .cpp where Impl is complete
Widget::Widget(const std::string& name)
    : pimpl_(std::make_unique<Impl>()) {
    pimpl_->name = name;
}

Widget::~Widget() = default;  // unique_ptr needs complete type here
Widget::Widget(Widget&&) noexcept = default;
Widget& Widget::operator=(Widget&&) noexcept = default;

void Widget::draw() const {
    pimpl_->internal_render();
}

void Widget::set_color(int r, int g, int b) {
    pimpl_->r = r;
    pimpl_->g = g;
    pimpl_->b = b;
}

```

### Q2: Show a Pimpl class that supports copying

**Answer:**

```cpp

// ─── config.h ───
#pragma once
#include <memory>
#include <string>

class Config {
public:
    Config();
    ~Config();
    Config(const Config&);             // Deep copy
    Config& operator=(const Config&);  // Deep copy
    Config(Config&&) noexcept;
    Config& operator=(Config&&) noexcept;

    void set(const std::string& key, const std::string& val);
    std::string get(const std::string& key) const;

private:
    struct Impl;
    std::unique_ptr<Impl> pimpl_;
};

// ─── config.cpp ───
#include "config.h"
#include <unordered_map>

struct Config::Impl {
    std::unordered_map<std::string, std::string> data;

    // Clone helper
    std::unique_ptr<Impl> clone() const {
        auto copy = std::make_unique<Impl>();
        copy->data = data;
        return copy;
    }
};

Config::Config() : pimpl_(std::make_unique<Impl>()) {}
Config::~Config() = default;
Config::Config(Config&&) noexcept = default;
Config& Config::operator=(Config&&) noexcept = default;

// Copy via deep clone
Config::Config(const Config& o)
    : pimpl_(o.pimpl_ ? o.pimpl_->clone() : nullptr) {}

Config& Config::operator=(const Config& o) {
    if (this != &o)
        pimpl_ = o.pimpl_ ? o.pimpl_->clone() : nullptr;
    return *this;
}

void Config::set(const std::string& key, const std::string& val) {
    pimpl_->data[key] = val;
}
std::string Config::get(const std::string& key) const {
    auto it = pimpl_->data.find(key);
    return it != pimpl_->data.end() ? it->second : "";
}

```

### Q3: Show fast Pimpl (stack-allocated) for performance-critical code

**Answer:**

```cpp

#include <cstddef>
#include <new>
#include <type_traits>
#include <utility>

// Fast Pimpl: allocate Impl in aligned stack buffer
class FastWidget {
public:
    FastWidget();
    ~FastWidget();
    FastWidget(const FastWidget&) = delete;
    void process();

private:
    struct Impl;
    // Buffer sized to hold Impl — update if Impl grows
    static constexpr size_t ImplSize = 128;
    static constexpr size_t ImplAlign = 8;
    alignas(ImplAlign) char storage_[ImplSize];

    Impl* impl() { return reinterpret_cast<Impl*>(storage_); }
    const Impl* impl() const { return reinterpret_cast<const Impl*>(storage_); }
};

// ─── fast_widget.cpp ───
#include <vector>
#include <string>
#include <iostream>

struct FastWidget::Impl {
    std::string name = "fast";
    std::vector<int> data;
};

FastWidget::FastWidget() {
    static_assert(sizeof(Impl) <= ImplSize, "Increase ImplSize!");
    static_assert(alignof(Impl) <= ImplAlign, "Increase ImplAlign!");
    new (storage_) Impl();  // Placement new
}

FastWidget::~FastWidget() {
    impl()->~Impl();  // Explicit destructor call
}

void FastWidget::process() {
    impl()->data.push_back(42);
    std::cout << impl()->name << ": " << impl()->data.size() << "\n";
}
// Zero heap allocations for Pimpl itself!

```

---

## Notes

- **Destructor must be declared in header, defined in .cpp** — `unique_ptr` needs complete type for deletion
- Move operations must also be defined in .cpp for the same reason
- Cost: one heap allocation per object + one pointer indirection per call
- Fast Pimpl avoids the heap but requires manually sizing the buffer — add a `static_assert` to catch size drift
- Pimpl is essential for shared library (`.so`/`.dll`) ABI stability — adding private members doesn't change class size
- Qt uses Pimpl ("d-pointer") extensively throughout its codebase
