# Use multiple inheritance safely with virtual inheritance and interfaces

**Category:** OOP Design

---

## Topic Overview

### The Diamond Problem

```cpp

         Animal
        /      \
    Mammal    Flyer
        \      /
        Bat        ← Two copies of Animal?

```

| MI Type | Use Case | Pitfalls |
| --- | --- | --- |
| Interface MI | Multiple abstract interfaces | None — **always safe** |
| Implementation MI | Reuse from multiple bases | Diamond, ambiguity |
| Virtual inheritance | Solve diamond problem | Performance cost, complexity |

### Safe Rules

```cpp

1. Multiple INTERFACES: always OK (pure virtual, no data)
2. Multiple bases with DATA: use virtual inheritance for diamond
3. Prefer composition over MI for code reuse
4. Never use MI just because you can — use it because you must

```

---

## Self-Assessment

### Q1: Show the diamond problem and fix it with virtual inheritance

**Answer:**

```cpp

#include <iostream>
#include <string>

// WITHOUT virtual inheritance: diamond problem
namespace broken {
    class Animal {
    public:
        std::string name;
        Animal(std::string n) : name(std::move(n)) {
            std::cout << "Animal(" << name << ") constructed\n";
        }
    };
    class Mammal : public Animal {
    public:
        Mammal(std::string n) : Animal(std::move(n)) {}
    };
    class Flyer : public Animal {
    public:
        Flyer(std::string n) : Animal(std::move(n)) {}
    };
    // class Bat : public Mammal, public Flyer {
    //     Bat() : Mammal("bat-m"), Flyer("bat-f") {}
    //     // bat.name is AMBIGUOUS! Two Animal subobjects
    // };
}

// WITH virtual inheritance: single Animal subobject
namespace fixed {
    class Animal {
    public:
        std::string name;
        Animal(std::string n = "") : name(std::move(n)) {
            std::cout << "Animal(" << name << ") constructed\n";
        }
        virtual ~Animal() = default;
    };
    class Mammal : virtual public Animal {
    public:
        Mammal(std::string n) : Animal(std::move(n)) {
            std::cout << "Mammal constructed\n";
        }
    };
    class Flyer : virtual public Animal {
    public:
        Flyer(std::string n) : Animal(std::move(n)) {
            std::cout << "Flyer constructed\n";
        }
    };
    class Bat : public Mammal, public Flyer {
    public:
        // IMPORTANT: Most-derived class must initialize virtual base
        Bat(std::string n)
            : Animal(n)        // Must explicitly init virtual base
            , Mammal(n)
            , Flyer(n) {}
    };
}

int main() {
    fixed::Bat bat("Bruce");
    std::cout << bat.name << "\n";  // Unambiguous! Single Animal
    return 0;
}

```

### Q2: Use multiple pure interfaces safely (the recommended pattern)

**Answer:**

```cpp

#include <iostream>
#include <memory>

// Pure interfaces: zero data, only virtual methods
class ISerializable {
public:
    virtual ~ISerializable() = default;
    virtual std::string serialize() const = 0;
};

class IRenderable {
public:
    virtual ~IRenderable() = default;
    virtual void render() const = 0;
};

class IClickable {
public:
    virtual ~IClickable() = default;
    virtual void on_click(int x, int y) = 0;
};

// Multiple interface inheritance: ALWAYS SAFE
class Button : public IRenderable, public IClickable, public ISerializable {
    std::string label_;
public:
    explicit Button(std::string label) : label_(std::move(label)) {}

    void render() const override {
        std::cout << "[" << label_ << "]\n";
    }
    void on_click(int x, int y) override {
        std::cout << label_ << " clicked at (" << x << "," << y << ")\n";
    }
    std::string serialize() const override {
        return "Button:" + label_;
    }
};

// Polymorphic usage through any interface
void draw(const IRenderable& r) { r.render(); }
void save(const ISerializable& s) { std::cout << s.serialize() << "\n"; }

int main() {
    Button btn("Submit");
    draw(btn);           // Uses IRenderable
    save(btn);           // Uses ISerializable
    btn.on_click(10, 20); // Uses IClickable
    return 0;
}

```

### Q3: Show mixin classes using MI for reusable behavior

**Answer:**

```cpp

#include <iostream>
#include <chrono>
#include <mutex>
#include <string>

// Mixin: adds timestamp tracking to any class
template<typename Base>
class Timestamped : public Base {
    using Clock = std::chrono::system_clock;
    Clock::time_point created_ = Clock::now();
    Clock::time_point modified_ = Clock::now();
public:
    using Base::Base;  // Inherit constructors

    void touch() { modified_ = Clock::now(); }
    auto age() const {
        return std::chrono::duration_cast<std::chrono::seconds>(
            Clock::now() - created_).count();
    }
};

// Mixin: adds thread-safety to any class
template<typename Base>
class ThreadSafe : public Base {
    mutable std::mutex mtx_;
public:
    using Base::Base;

    template<typename F>
    auto with_lock(F&& f) {
        std::lock_guard lk(mtx_);
        return f(static_cast<Base&>(*this));
    }
    template<typename F>
    auto with_lock(F&& f) const {
        std::lock_guard lk(mtx_);
        return f(static_cast<const Base&>(*this));
    }
};

// Base class
class Document {
    std::string content_;
public:
    Document() = default;
    explicit Document(std::string c) : content_(std::move(c)) {}
    void set_content(std::string c) { content_ = std::move(c); }
    const std::string& content() const { return content_; }
};

// Compose mixins via inheritance chain
using SafeTimedDoc = ThreadSafe<Timestamped<Document>>;

int main() {
    SafeTimedDoc doc("Initial content");

    doc.with_lock([](Document& d) {
        d.set_content("Updated");
    });

    doc.touch();
    std::cout << "Age: " << doc.age() << "s\n";

    doc.with_lock([](const Document& d) {
        std::cout << d.content() << "\n";
    });
    return 0;
}

```

---

## Notes

- **Safe MI rule:** multiple *interfaces* (pure virtual, no data) is always fine
- Virtual inheritance adds ~8 bytes overhead per object and slightly slower dispatch
- The **most-derived class** must explicitly initialize virtual bases in its constructor
- Prefer **composition** or **CRTP mixins** over implementation-level multiple inheritance
- Mixin pattern via templates (`ThreadSafe<Timestamped<Base>>`) is the modern C++ approach to MI
- If you need diamond inheritance, it's often a sign the design should be refactored
