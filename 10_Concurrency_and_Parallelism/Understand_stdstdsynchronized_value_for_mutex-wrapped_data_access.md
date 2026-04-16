# Understand std::synchronized_value for mutex-wrapped data access

**Category:** Concurrency and Parallelism  
**Standard:** C++26 (proposed)  
**Reference:** <https://wg21.link/P0290>  

---

## Topic Overview

`synchronized_value<T>` bundles a value with its mutex, ensuring all accesses go through the lock. This prevents the common bug of accessing shared data without locking.

### The Problem

```cpp

#include <mutex>
#include <string>

// Classic pattern: value + mutex — easy to forget the lock
struct Shared {
    std::mutex mtx;
    std::string data;
};

void unsafe(Shared& s) {
    s.data = "oops";  // BUG: forgot to lock s.mtx!
}

```

### synchronized_value Pattern

```cpp

#include <mutex>
#include <string>
#include <iostream>

// Implementation (until standardized):
template<typename T>
class synchronized_value {
    T value_;
    mutable std::mutex mtx_;

public:
    template<typename... Args>
    synchronized_value(Args&&... args) : value_(std::forward<Args>(args)...) {}

    // Apply a function under the lock
    template<typename F>
    auto apply(F&& f) {
        std::lock_guard lock(mtx_);
        return std::forward<F>(f)(value_);
    }

    template<typename F>
    auto apply(F&& f) const {
        std::lock_guard lock(mtx_);
        return std::forward<F>(f)(value_);
    }
};

// Usage: impossible to access without locking
synchronized_value<std::string> shared_data("initial");

void safe_update() {
    shared_data.apply([](std::string& s) {
        s += " modified";
    });
}

void safe_read() {
    auto copy = shared_data.apply([](const std::string& s) {
        return s;  // Copy under lock
    });
    std::cout << copy << "\n";
}

```

---

## Self-Assessment

### Q1: How does synchronized_value prevent data races

The value is private — the only way to access it is through `apply()`, which takes the lock first. There's no way to get a reference to the underlying data without holding the mutex.

### Q2: What about multi-value transactions

```cpp

// For locking multiple synchronized_values atomically:
template<typename F, typename... SVs>
auto apply_all(F&& f, SVs&... svs) {
    std::scoped_lock lock(svs.mutex()...);
    return f(svs.unsafe_value()...);
}

```

### Q3: Compare with Rust's Mutex<T>

Rust's `Mutex<T>` works the same way: the data is only accessible through a lock guard. `MutexGuard<T>` provides `Deref` to `T`. C++'s `synchronized_value` provides `apply(F)` instead. Both prevent unlocked access at the API level.

---

## Notes

- P0290 proposes `std::synchronized_value` for the standard library.
- `boost::synchronized_value` exists in Boost.Thread.
- The pattern is also called "monitor" in concurrency literature.
- Use `apply()` returning a value (not a reference) to avoid dangling references outside the lock.
