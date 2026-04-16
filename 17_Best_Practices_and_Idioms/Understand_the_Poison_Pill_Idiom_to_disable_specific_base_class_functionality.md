# Understand the Poison Pill Idiom to disable specific base class functionality

**Category:** Best Practices & Idioms  
**Item:** #409  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines>  

---

## Topic Overview

The **Poison Pill Idiom** uses `= delete` to prevent specific inherited or overloaded functions from being called. This is distinct from the "poison pill" in ranges/ADL (which uses `= delete` on free functions to prevent unwanted ADL).

### Two Forms

```cpp

1. Derived class disables an inherited method:

   class Derived : public Base {
       void unwanted_method() = delete;  // "poison" this method
   };

2. ADL poison pill (ranges library):

   void begin(auto&&) = delete;  // prevent ADL from finding wrong begin()

```

---

## Self-Assessment

### Q1: Use `= delete` in a derived class to disable an inherited public method

```cpp

#include <iostream>
#include <string>

class Collection {
public:
    void add(const std::string& item) {
        std::cout << "Added: " << item << '\n';
    }
    void remove(const std::string& item) {
        std::cout << "Removed: " << item << '\n';
    }
    void clear() {
        std::cout << "Cleared all\n";
    }
};

// AppendOnlyLog: can add, but NOT remove or clear
class AppendOnlyLog : public Collection {
public:
    void remove(const std::string&) = delete;  // poisoned!
    void clear() = delete;                      // poisoned!
};

int main() {
    AppendOnlyLog log;
    log.add("entry1");            // OK
    log.add("entry2");            // OK
    // log.remove("entry1");      // ERROR: use of deleted function
    // log.clear();               // ERROR: use of deleted function
    std::cout << "Append-only enforced at compile time!\n";
}
// Expected output:
// Added: entry1
// Added: entry2
// Append-only enforced at compile time!

```

### Q2: Explain why deleting via derived doesn't work through a base pointer

```cpp

#include <iostream>
#include <string>

class Collection {
public:
    virtual ~Collection() = default;
    void add(const std::string& item) { std::cout << "Added: " << item << '\n'; }
    void remove(const std::string& item) { std::cout << "Removed: " << item << '\n'; }
};

class AppendOnly : public Collection {
public:
    void remove(const std::string&) = delete;  // deleted in derived
};

int main() {
    AppendOnly log;
    // log.remove("x");  // ERROR: deleted — good!

    // BUT: through base pointer, it's still callable!
    Collection* base_ptr = &log;
    base_ptr->remove("x");  // COMPILES AND RUNS! Delete is bypassed!
    // The delete only affects static type (AppendOnly), not dynamic dispatch

    std::cout << "Base pointer bypassed the delete!\n";
}
// Expected output:
// Removed: x
// Base pointer bypassed the delete!
//
// LESSON: = delete on a non-virtual function in derived class
// only affects calls through the derived type.
// Base pointer/reference still sees the base version.

```

### Q3: Show the correct approach using private inheritance or composition

```cpp

#include <iostream>
#include <string>
#include <vector>

// Approach 1: Private inheritance + using declarations
class CollectionBase {
public:
    void add(const std::string& item) { items_.push_back(item); }
    void remove(const std::string& item) {
        items_.erase(std::remove(items_.begin(), items_.end(), item), items_.end());
    }
    void clear() { items_.clear(); }
    void print() const {
        for (const auto& s : items_) std::cout << s << ' ';
        std::cout << '\n';
    }
protected:
    std::vector<std::string> items_;
};

// Private inheritance: nothing is public by default
class AppendOnlyV1 : private CollectionBase {
public:
    using CollectionBase::add;    // expose only add
    using CollectionBase::print;  // and print
    // remove() and clear() are NOT exposed
};

// Approach 2: Composition (preferred)
class AppendOnlyV2 {
    std::vector<std::string> items_;
public:
    void add(const std::string& item) { items_.push_back(item); }
    void print() const {
        for (const auto& s : items_) std::cout << s << ' ';
        std::cout << '\n';
    }
    // remove() and clear() simply don't exist
};

int main() {
    AppendOnlyV1 log1;
    log1.add("a");
    log1.add("b");
    // log1.remove("a");  // ERROR: inaccessible
    // log1.clear();       // ERROR: inaccessible
    log1.print();

    AppendOnlyV2 log2;
    log2.add("x");
    log2.add("y");
    // log2.remove("x");  // ERROR: no such function
    log2.print();
}
// Expected output:
// a b
// x y

```

---

## Notes

- `= delete` is a compile-time mechanism — it only affects the static type.
- For true runtime restriction, use virtual methods with proper interface design.
- Composition is almost always better than inheritance when you want to restrict the interface.
- The ranges library uses "poison pills" differently: `void begin(auto&&) = delete;` prevents unwanted ADL matches.
