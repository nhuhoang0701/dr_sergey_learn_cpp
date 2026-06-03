# Understand std::synchronized_value for mutex-wrapped data access

**Category:** Concurrency and Parallelism  
**Standard:** C++26 (proposed)  
**Reference:** <https://wg21.link/P0290>  

---

## Topic Overview

`synchronized_value<T>` bundles a value together with its mutex, so every access to that value automatically goes through the lock. This sounds simple, but it solves a genuinely annoying class of bug: forgetting to acquire the mutex before touching the protected data. Because the value is private and the only way in is through `apply()`, you literally cannot forget the lock - the API won't let you.

### The Problem

The classic approach to thread-safe data is to put a mutex next to your value in a struct. It works, but there's nothing stopping you from reaching past the mutex and directly accessing the data. Here's what that mistake looks like:

```cpp
#include <mutex>
#include <string>

// Classic pattern: value + mutex - easy to forget the lock
struct Shared {
    std::mutex mtx;
    std::string data;
};

void unsafe(Shared& s) {
    s.data = "oops";  // BUG: forgot to lock s.mtx!
}
```

The compiler won't catch this. The mutex and the data are just two independent members, and nothing enforces that you use one before the other.

### synchronized_value Pattern

`synchronized_value` wraps the value so tightly that you can only interact with it by passing a lambda (or callable) to `apply()`. The implementation takes the lock, calls your function with a reference to the inner value, and releases the lock when the function returns. If you never get a raw reference to the value, you can't accidentally bypass the lock.

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

Notice that `safe_read()` returns a *copy*, not a reference. That's intentional - if `apply()` returned a reference, you could hold onto it after the lock is released, which would defeat the whole point.

---

## Self-Assessment

### Q1: How does synchronized_value prevent data races

The value is private - the only way to access it is through `apply()`, which takes the lock first. There's no way to get a reference to the underlying data without holding the mutex. The API design makes the safe path the only path.

### Q2: What about multi-value transactions

Sometimes you need to update two synchronized values atomically - for example, transferring a balance between two accounts. Locking one at a time won't work because another thread could observe an inconsistent intermediate state between the two locks. The solution is to acquire both locks at once with `std::scoped_lock`, which handles deadlock avoidance automatically:

```cpp
// For locking multiple synchronized_values atomically:
template<typename F, typename... SVs>
auto apply_all(F&& f, SVs&... svs) {
    std::scoped_lock lock(svs.mutex()...);
    return f(svs.unsafe_value()...);
}
```

### Q3: Compare with Rust's Mutex<T>

Rust's `Mutex<T>` works the same way: the data is only accessible through a lock guard. `MutexGuard<T>` provides `Deref` to `T`. C++'s `synchronized_value` provides `apply(F)` instead. Both prevent unlocked access at the API level. The philosophical goal is identical - if you want the data, you have to go through the lock, and the type system enforces that.

---

## Notes

- P0290 proposes `std::synchronized_value` for the standard library.
- `boost::synchronized_value` exists in Boost.Thread if you need it before C++26.
- The pattern is also called a "monitor" in concurrency literature - a classic synchronization structure that pairs data with its own lock.
- Use `apply()` returning a value (not a reference) to avoid dangling references outside the lock.
