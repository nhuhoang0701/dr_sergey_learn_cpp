# Understand std::owner_less for comparing weak_ptr in ordered containers

**Category:** Memory and Ownership  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/memory/owner_less>  

---

## Topic Overview

`std::weak_ptr` doesn't support `operator<` because pointer comparison depends on the managed object's address, which may change or become null. `std::owner_less` compares by ownership (control block), enabling ordered containers.

### The Problem

```cpp

#include <memory>
#include <set>

auto sp1 = std::make_shared<int>(42);
auto sp2 = sp1;
std::weak_ptr<int> wp1 = sp1;
std::weak_ptr<int> wp2 = sp2;

// wp1 and wp2 both point to the same object
// But std::weak_ptr has no operator<

// std::set<std::weak_ptr<int>> s;  // Won't compile — no operator<

```

### The Solution: owner_less

```cpp

#include <memory>
#include <set>
#include <iostream>

int main() {
    // owner_less compares by control block address, not object address
    std::set<std::weak_ptr<int>, std::owner_less<std::weak_ptr<int>>> tracked;

    auto sp1 = std::make_shared<int>(1);
    auto sp2 = std::make_shared<int>(2);
    auto sp3 = sp1;  // Same ownership as sp1

    tracked.insert(sp1);
    tracked.insert(sp2);
    tracked.insert(sp3);  // Same ownership as sp1 — NOT inserted (duplicate)

    std::cout << tracked.size() << "\n";  // 2 (not 3)

    // owner_less<void> works with any smart pointer type (C++17):
    std::set<std::weak_ptr<int>, std::owner_less<>> modern_set;
}

```

### Use Case: Observer Pattern

```cpp

#include <memory>
#include <set>
#include <vector>
#include <algorithm>

class Subject {
    std::set<std::weak_ptr<Observer>, std::owner_less<>> observers_;

public:
    void subscribe(std::weak_ptr<Observer> obs) {
        observers_.insert(std::move(obs));
    }

    void notify() {
        // Remove expired observers and notify live ones
        for (auto it = observers_.begin(); it != observers_.end(); ) {
            if (auto sp = it->lock()) {
                sp->on_event();
                ++it;
            } else {
                it = observers_.erase(it);  // Expired
            }
        }
    }
};

```

---

## Self-Assessment

### Q1: Why can't weak_ptr use regular pointer comparison

A `weak_ptr` may be expired (the managed object deleted). Comparing by object address would compare dangling pointers (UB). `owner_less` compares by control block address, which remains valid even after the managed object is destroyed.

### Q2: What does `owner_less<void>` provide

`owner_less<void>` (C++17) is a transparent comparator that works across `shared_ptr<T>`, `weak_ptr<T>`, and mixed comparisons. It provides `operator()` overloads for all combinations.

### Q3: When would you use owner_before directly

`shared_ptr::owner_before()` and `weak_ptr::owner_before()` are member functions that do the same comparison. Use them for ad-hoc comparisons; use `owner_less` as a comparator for containers.

---

## Notes

- `owner_less` is required for `std::set<weak_ptr<T>>` and `std::map<weak_ptr<T>, V>`.
- Prefer `owner_less<>` (void specialization) for maximum flexibility.
- The control block outlives the managed object — `owner_less` remains valid after expiry.
- Key use cases: observer patterns, caches, and dependency tracking.
