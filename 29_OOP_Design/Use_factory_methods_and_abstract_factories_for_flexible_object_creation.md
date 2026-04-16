# Use factory methods and abstract factories for flexible object creation

**Category:** OOP Design

---

## Topic Overview

| Pattern | Purpose | Returns | Extensibility |
| --- | --- | --- | --- |
| **Factory method** | Create one product | `unique_ptr<Base>` | Override in derived class |
| **Static factory** | Named constructors | Value or `unique_ptr` | Single class |
| **Abstract factory** | Create family of related products | Multiple `unique_ptr<Base>` | New families via new factory |

### When to Use

```cpp

Factory method:      Type determined by subclass or parameter
Static factory:      Multiple construction modes for one type
Abstract factory:    Families of related objects that must match
                     (e.g., UI widgets for different platforms)

```

---

## Self-Assessment

### Q1: Implement static factory methods and a parameterized factory

**Answer:**

```cpp

#include <memory>
#include <string>
#include <cmath>
#include <stdexcept>
#include <iostream>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string name() const = 0;
};

class Circle : public Shape {
    double r_;
public:
    explicit Circle(double r) : r_(r) {}
    double area() const override { return M_PI * r_ * r_; }
    std::string name() const override { return "Circle"; }

    // Static factory: named constructors
    static Circle unit() { return Circle(1.0); }
    static Circle from_area(double a) { return Circle(std::sqrt(a / M_PI)); }
};

class Rect : public Shape {
    double w_, h_;
public:
    Rect(double w, double h) : w_(w), h_(h) {}
    double area() const override { return w_ * h_; }
    std::string name() const override { return "Rect"; }

    static Rect square(double side) { return Rect(side, side); }
};

// Parameterized factory function
std::unique_ptr<Shape> create_shape(const std::string& type, double a, double b = 0) {
    if (type == "circle") return std::make_unique<Circle>(a);
    if (type == "rect")   return std::make_unique<Rect>(a, b);
    throw std::invalid_argument("Unknown shape: " + type);
}

int main() {
    auto c = Circle::from_area(100.0);  // Named constructor
    auto s = Rect::square(5.0);         // Named constructor
    auto shape = create_shape("circle", 3.0);  // Parameterized factory
    std::cout << shape->name() << ": " << shape->area() << "\n";
    return 0;
}

```

### Q2: Implement an Abstract Factory for cross-platform UI

**Answer:**

```cpp

#include <memory>
#include <string>
#include <iostream>

// Abstract products
class Button {
public:
    virtual ~Button() = default;
    virtual void render() const = 0;
};

class TextBox {
public:
    virtual ~TextBox() = default;
    virtual void render() const = 0;
};

// Concrete products: Windows
class WinButton : public Button {
public:
    void render() const override { std::cout << "[Win Button]\n"; }
};
class WinTextBox : public TextBox {
public:
    void render() const override { std::cout << "[Win TextBox]\n"; }
};

// Concrete products: Linux
class LinuxButton : public Button {
public:
    void render() const override { std::cout << "[Linux Button]\n"; }
};
class LinuxTextBox : public TextBox {
public:
    void render() const override { std::cout << "[Linux TextBox]\n"; }
};

// Abstract Factory
class UIFactory {
public:
    virtual ~UIFactory() = default;
    virtual std::unique_ptr<Button> create_button() const = 0;
    virtual std::unique_ptr<TextBox> create_textbox() const = 0;
};

class WinFactory : public UIFactory {
public:
    std::unique_ptr<Button> create_button() const override {
        return std::make_unique<WinButton>();
    }
    std::unique_ptr<TextBox> create_textbox() const override {
        return std::make_unique<WinTextBox>();
    }
};

class LinuxFactory : public UIFactory {
public:
    std::unique_ptr<Button> create_button() const override {
        return std::make_unique<LinuxButton>();
    }
    std::unique_ptr<TextBox> create_textbox() const override {
        return std::make_unique<LinuxTextBox>();
    }
};

// Client code works with any factory
void build_ui(const UIFactory& factory) {
    auto btn = factory.create_button();
    auto txt = factory.create_textbox();
    btn->render();
    txt->render();
}

int main() {
    WinFactory wf;
    LinuxFactory lf;
    build_ui(wf);  // Consistent Windows UI
    build_ui(lf);  // Consistent Linux UI
    return 0;
}

```

### Q3: Show a self-registering factory using a map of creators

**Answer:**

```cpp

#include <memory>
#include <string>
#include <unordered_map>
#include <functional>
#include <stdexcept>
#include <iostream>

class Plugin {
public:
    virtual ~Plugin() = default;
    virtual void execute() = 0;
};

// Self-registering factory
class PluginFactory {
public:
    using Creator = std::function<std::unique_ptr<Plugin>()>;

    static PluginFactory& instance() {
        static PluginFactory f;
        return f;
    }

    void register_plugin(const std::string& name, Creator creator) {
        creators_[name] = std::move(creator);
    }

    std::unique_ptr<Plugin> create(const std::string& name) const {
        auto it = creators_.find(name);
        if (it == creators_.end())
            throw std::runtime_error("Unknown plugin: " + name);
        return it->second();
    }

private:
    PluginFactory() = default;
    std::unordered_map<std::string, Creator> creators_;
};

// Auto-registration helper
template<typename T>
struct RegisterPlugin {
    explicit RegisterPlugin(const std::string& name) {
        PluginFactory::instance().register_plugin(
            name, [] { return std::make_unique<T>(); });
    }
};

// Plugins self-register at static init time
class Compressor : public Plugin {
    static RegisterPlugin<Compressor> reg_;
public:
    void execute() override { std::cout << "Compressing...\n"; }
};
RegisterPlugin<Compressor> Compressor::reg_{"compressor"};

class Encryptor : public Plugin {
    static RegisterPlugin<Encryptor> reg_;
public:
    void execute() override { std::cout << "Encrypting...\n"; }
};
RegisterPlugin<Encryptor> Encryptor::reg_{"encryptor"};

int main() {
    auto p = PluginFactory::instance().create("compressor");
    p->execute();  // "Compressing..."
    return 0;
}

```

---

## Notes

- **Static factory methods** are the simplest — use for named constructors (`from_string()`, `default_config()`)
- Parameterized factories use a string/enum key — extensible with a registration map
- Abstract factories enforce **consistency** across product families
- Self-registering factories avoid a central switch statement — new types add themselves
- Always return `unique_ptr` from factories — caller controls ownership
- Prefer free factory functions or static methods over the GoF virtual factory method pattern in modern C++
