# Design exception-safe classes with strong and basic guarantees

**Category:** OOP Design

---

## Topic Overview

Exception safety guarantees are contractual promises about class state when an operation throws:

| Guarantee | Promise | State After Throw | Example |
| --- | --- | --- | --- |
| **No-throw** | Never throws | N/A | `swap()`, destructors, `noexcept` move |
| **Strong** | Commit-or-rollback | Original state preserved | `std::vector::push_back` |
| **Basic** | No leaks, valid state | Valid but unspecified | `std::vector::insert` (partial) |
| **None** | Anything goes | Possibly corrupted | **Avoid this** |

### Key Techniques

```cpp

Strong guarantee strategy:

1. Perform all throwing work on temporaries / copies
2. Commit results with non-throwing operations (swap/move)
3. Never modify observable state before all throwing ops complete

```

---

## Self-Assessment

### Q1: Write a class with the strong exception guarantee using copy-and-swap

**Answer:**

```cpp

#include <algorithm>
#include <cstring>
#include <stdexcept>
#include <utility>

class DynamicBuffer {
    char* data_;
    size_t size_;
    size_t capacity_;
public:
    explicit DynamicBuffer(size_t cap = 0)
        : data_(cap ? new char[cap]{} : nullptr)
        , size_(0), capacity_(cap) {}

    ~DynamicBuffer() { delete[] data_; }

    // Copy constructor — can throw (allocates)
    DynamicBuffer(const DynamicBuffer& o)
        : data_(o.size_ ? new char[o.capacity_] : nullptr)
        , size_(o.size_), capacity_(o.capacity_) {
        std::memcpy(data_, o.data_, size_);
    }

    // Move constructor — noexcept!
    DynamicBuffer(DynamicBuffer&& o) noexcept
        : data_(std::exchange(o.data_, nullptr))
        , size_(std::exchange(o.size_, 0))
        , capacity_(std::exchange(o.capacity_, 0)) {}

    // Unified assignment via copy-and-swap → STRONG guarantee
    DynamicBuffer& operator=(DynamicBuffer o) noexcept {
        swap(*this, o);  // o dies with old data
        return *this;
    }

    friend void swap(DynamicBuffer& a, DynamicBuffer& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
        swap(a.capacity_, b.capacity_);
    }

    // append() with STRONG guarantee
    void append(const char* src, size_t n) {
        if (size_ + n > capacity_) {
            // Step 1: All throwing work on a temporary
            DynamicBuffer tmp(std::max(capacity_ * 2, size_ + n));
            std::memcpy(tmp.data_, data_, size_);
            std::memcpy(tmp.data_ + size_, src, n);
            tmp.size_ = size_ + n;
            // Step 2: Commit with noexcept swap
            swap(*this, tmp);  // If we get here, commit
        } else {
            // No allocation needed — basic guarantee is sufficient
            std::memcpy(data_ + size_, src, n);
            size_ += n;
        }
    }

    size_t size() const noexcept { return size_; }
};

```

### Q2: Show the "do all work, then swap" idiom for multi-member updates

**Answer:**

```cpp

#include <string>
#include <vector>
#include <mutex>

class UserProfile {
    std::string name_;
    std::string email_;
    std::vector<std::string> roles_;
    mutable std::mutex mtx_;
public:
    // STRONG guarantee: either all fields update or none do
    void update(const std::string& name, const std::string& email,
                const std::vector<std::string>& roles) {
        // Phase 1: all potentially-throwing copies happen here
        std::string new_name = name;          // may throw
        std::string new_email = email;        // may throw
        std::vector<std::string> new_roles = roles;  // may throw

        // Phase 2: commit with non-throwing operations
        std::lock_guard lk(mtx_);
        name_  = std::move(new_name);    // noexcept (move)
        email_ = std::move(new_email);   // noexcept
        roles_ = std::move(new_roles);   // noexcept
    }
    // If Phase 1 throws, object untouched → strong guarantee
};

```

### Q3: Implement a transactional container with rollback on failure

**Answer:**

```cpp

#include <vector>
#include <functional>
#include <stdexcept>
#include <iostream>

// ScopeGuard for automatic rollback
class ScopeGuard {
    std::function<void()> rollback_;
    bool committed_ = false;
public:
    explicit ScopeGuard(std::function<void()> fn) : rollback_(std::move(fn)) {}
    ~ScopeGuard() { if (!committed_) rollback_(); }
    void commit() noexcept { committed_ = true; }
};

// Transactional batch insert with strong guarantee
template<typename T>
class TransactionalVector {
    std::vector<T> data_;
public:
    // Insert multiple items — all succeed or all roll back
    void batch_insert(const std::vector<T>& items) {
        auto original_size = data_.size();
        ScopeGuard guard([&] {
            data_.resize(original_size);  // rollback
        });

        for (const auto& item : items) {
            data_.push_back(item);  // may throw
        }

        guard.commit();  // all succeeded, don't rollback
    }

    size_t size() const { return data_.size(); }
};

int main() {
    TransactionalVector<std::string> tv;
    try {
        tv.batch_insert({"alpha", "beta", "gamma"});
        std::cout << "After success: " << tv.size() << "\n";  // 3
    } catch (...) {}
    return 0;
}

```

---

## Notes

- **Destructors must never throw** — they're implicitly `noexcept` since C++11
- Mark move constructors/assignment `noexcept` for strong guarantees in containers
- The copy-and-swap idiom gives strong guarantee "for free" but costs one extra allocation
- For multi-member updates: do all throwing work first, then commit with moves/swaps
- `std::vector::push_back` provides strong guarantee; `insert` in the middle may only provide basic
- Use ScopeGuard / RAII for multi-step operations needing rollback
