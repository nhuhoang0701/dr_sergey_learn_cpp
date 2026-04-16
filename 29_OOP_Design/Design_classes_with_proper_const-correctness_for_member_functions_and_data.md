# Design classes with proper const-correctness for member functions and data

**Category:** OOP Design

---

## Topic Overview

| Rule | Meaning |
| --- | --- |
| `const` member function | Promises not to modify observable state |
| `const` data member | Set once in constructor, never changes |
| `mutable` member | Can change even in `const` context (caching, mutexes) |
| `const T&` return | Prevents caller from modifying internal state |

### Const Correctness Principle

```cpp

"const" is a contract with callers:

  - If a method is const, calling it won't change object state
  - If a reference is const, the object won't be modified through it
  - The compiler ENFORCES this contract

```

---

## Self-Assessment

### Q1: Design a class with proper const member function overloads

**Answer:**

```cpp

#include <vector>
#include <string>
#include <stdexcept>
#include <iostream>

class Document {
public:
    void add_line(std::string line) {
        lines_.push_back(std::move(line));
    }

    // Const overload: returns const reference
    const std::string& operator[](size_t idx) const {
        if (idx >= lines_.size())
            throw std::out_of_range("Index out of range");
        return lines_[idx];
    }

    // Non-const overload: returns mutable reference
    std::string& operator[](size_t idx) {
        // Scott Meyers' trick: call const version to avoid duplication
        return const_cast<std::string&>(
            static_cast<const Document&>(*this)[idx]);
    }

    // Always const: doesn't modify state
    size_t size() const noexcept { return lines_.size(); }
    bool empty() const noexcept { return lines_.empty(); }

    // Return const reference to prevent modification of internals
    const std::vector<std::string>& lines() const noexcept { return lines_; }

private:
    std::vector<std::string> lines_;
};

void print_doc(const Document& doc) {
    // Can only call const member functions
    for (size_t i = 0; i < doc.size(); ++i)
        std::cout << doc[i] << "\n";  // Calls const operator[]
    // doc[0] = "modified";  // ERROR: returns const reference
}

int main() {
    Document doc;
    doc.add_line("Hello");
    doc[0] = "Modified";  // OK: calls non-const operator[]
    print_doc(doc);        // Uses const overloads
    return 0;
}

```

### Q2: Show proper use of mutable for caches and synchronization

**Answer:**

```cpp

#include <string>
#include <vector>
#include <numeric>
#include <mutex>
#include <optional>

class Statistics {
public:
    explicit Statistics(std::vector<double> data)
        : data_(std::move(data)) {}

    // Logically const — but caches result
    double mean() const {
        std::lock_guard lock(mtx_);  // mutable mutex
        if (!cached_mean_) {
            double sum = std::accumulate(data_.begin(), data_.end(), 0.0);
            cached_mean_ = sum / static_cast<double>(data_.size());
        }
        return *cached_mean_;
    }

    // Non-const: invalidates cache
    void add_sample(double val) {
        std::lock_guard lock(mtx_);
        data_.push_back(val);
        cached_mean_.reset();  // Invalidate
    }

    size_t count() const noexcept { return data_.size(); }

private:
    std::vector<double> data_;
    // mutable: changed in const methods for caching/locking
    mutable std::mutex mtx_;
    mutable std::optional<double> cached_mean_;
};
// mutable is ONLY for members that don't affect observable state

```

### Q3: Demonstrate const propagation with smart pointers and containers

**Answer:**

```cpp

#include <memory>
#include <vector>
#include <iostream>

class Config {
    int value_ = 42;
public:
    int get() const { return value_; }
    void set(int v) { value_ = v; }
};

class Service {
public:
    explicit Service(std::shared_ptr<Config> cfg)
        : cfg_(std::move(cfg)) {}

    // PROBLEM: raw pointer/shared_ptr doesn't propagate const
    void bad_const_method() const {
        cfg_->set(99);  // Compiles! shared_ptr<Config> not const-propagated
    }

    // SOLUTION 1: Expose only const interface
    int get_value() const {
        return cfg_->get();  // Only call const methods
    }

private:
    std::shared_ptr<Config> cfg_;
};

// SOLUTION 2: std::experimental::propagate_const (or write your own)
template<typename Ptr>
class PropagateConst {
    Ptr ptr_;
public:
    PropagateConst(Ptr p) : ptr_(std::move(p)) {}

    auto* operator->() { return &*ptr_; }
    const auto* operator->() const { return &*ptr_; }  // Forces const!

    auto& operator*() { return *ptr_; }
    const auto& operator*() const { return *ptr_; }
};

class BetterService {
    PropagateConst<std::unique_ptr<Config>> cfg_;
public:
    BetterService() : cfg_(std::make_unique<Config>()) {}

    void modify() { cfg_->set(99); }      // OK: non-const
    int read() const { return cfg_->get(); }  // OK: const
    // void bad() const { cfg_->set(1); }  // ERROR: const propagated!
};

int main() {
    BetterService svc;
    svc.modify();
    std::cout << svc.read() << "\n";
    return 0;
}

```

---

## Notes

- **Mark every method that doesn't modify state as `const`** — this is not optional
- `mutable` is acceptable for: mutexes, caches, debug counters — NOT for real state
- `const` on smart pointers doesn't propagate: `const unique_ptr<T>` → pointer is const, `T` is not
- Use `propagate_const` wrapper for Pimpl and owned-pointer members
- Returning `const T&` prevents callers from modifying internals; returning `T` gives them a copy
- The compiler can optimize through `const` — wrong use of `const_cast` is undefined behavior
