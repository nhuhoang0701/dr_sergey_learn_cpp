# Design class invariants and validate them with assertions

**Category:** OOP Design

---

## Topic Overview

A **class invariant** is a condition that must always hold true between public method calls:

```cpp

Constructor → establishes invariant
  ↓
Public method → may temporarily break invariant
  ↓                 during execution, but MUST
Method returns → restore invariant before returning
  ↓
Destructor → invariant holds for the last time

```

| Assertion Type | When | Cost | Production |
| --- | --- | --- | --- |
| `assert()` | Debug only | Zero in release | Compiled out |
| `static_assert` | Compile time | **Zero** | Always |
| Exceptions | Runtime | Non-zero | Active |
| Contracts (C++26) | Configurable | Configurable | Configurable |

---

## Self-Assessment

### Q1: Design a class with documented and enforced invariants

**Answer:**

```cpp

#include <vector>
#include <cassert>
#include <algorithm>
#include <stdexcept>
#include <iostream>

// Invariants:
// 1. data_ is always sorted
// 2. data_ contains no duplicates
// 3. size() <= max_capacity_
class SortedSet {
    std::vector<int> data_;
    size_t max_capacity_;

    // Private invariant checker
    bool check_invariant() const {
        // Sorted?
        for (size_t i = 1; i < data_.size(); ++i)
            if (data_[i-1] >= data_[i]) return false;
        // Capacity?
        return data_.size() <= max_capacity_;
    }

public:
    explicit SortedSet(size_t max_cap)
        : max_capacity_(max_cap) {
        assert(max_cap > 0 && "Capacity must be positive");
        assert(check_invariant());
    }

    bool insert(int value) {
        assert(check_invariant());  // Pre-condition

        if (data_.size() >= max_capacity_)
            return false;

        auto it = std::lower_bound(data_.begin(), data_.end(), value);
        if (it != data_.end() && *it == value)
            return false;  // Duplicate

        data_.insert(it, value);

        assert(check_invariant());  // Post-condition
        return true;
    }

    bool contains(int value) const {
        return std::binary_search(data_.begin(), data_.end(), value);
    }

    bool remove(int value) {
        assert(check_invariant());
        auto it = std::lower_bound(data_.begin(), data_.end(), value);
        if (it == data_.end() || *it != value) return false;
        data_.erase(it);
        assert(check_invariant());
        return true;
    }

    size_t size() const { return data_.size(); }
};

int main() {
    SortedSet s(5);
    s.insert(3); s.insert(1); s.insert(2);
    std::cout << s.contains(2) << "\n";  // true
    std::cout << s.insert(2) << "\n";    // false (duplicate)
    return 0;
}

```

### Q2: Use defensive programming for untrusted inputs

**Answer:**

```cpp

#include <string>
#include <stdexcept>
#include <cassert>
#include <iostream>

class BankAccount {
    std::string owner_;
    double balance_;  // Invariant: balance_ >= 0

    void validate_invariant() const {
        assert(balance_ >= 0.0 && "Balance invariant violated!");
        assert(!owner_.empty() && "Owner invariant violated!");
    }

public:
    // BOUNDARY: validate untrusted input with exceptions
    BankAccount(const std::string& owner, double initial_balance) {
        if (owner.empty())
            throw std::invalid_argument("Owner name required");
        if (initial_balance < 0)
            throw std::invalid_argument("Initial balance must be >= 0");

        owner_ = owner;
        balance_ = initial_balance;
        validate_invariant();  // Established!
    }

    // BOUNDARY: public API validates with exceptions
    void deposit(double amount) {
        if (amount <= 0)
            throw std::invalid_argument("Deposit must be positive");

        balance_ += amount;  // Can't break invariant (adding positive)
        validate_invariant();
    }

    void withdraw(double amount) {
        if (amount <= 0)
            throw std::invalid_argument("Withdrawal must be positive");
        if (amount > balance_)
            throw std::runtime_error("Insufficient funds");

        balance_ -= amount;
        validate_invariant();
    }

    // INTERNAL: assert for programmer errors
    void apply_interest(double rate) {
        assert(rate >= 0.0 && rate <= 1.0 && "Rate must be 0..1");
        balance_ *= (1.0 + rate);
        validate_invariant();
    }

    double balance() const { return balance_; }
};

int main() {
    BankAccount acc("Alice", 1000);
    acc.deposit(500);
    acc.withdraw(200);
    acc.apply_interest(0.05);  // 5% interest
    std::cout << "Balance: " << acc.balance() << "\n";
    return 0;
}

```

### Q3: Show compile-time invariant checking

**Answer:**

```cpp

#include <cstddef>
#include <array>
#include <type_traits>
#include <iostream>

// Compile-time invariant: buffer size must be power of two
template<size_t N>
class RingBuffer {
    static_assert(N > 0, "Buffer size must be positive");
    static_assert((N & (N - 1)) == 0, "Buffer size must be power of 2");

    std::array<int, N> data_{};
    size_t head_ = 0, tail_ = 0;

public:
    void push(int val) {
        data_[head_ & (N - 1)] = val;  // Fast modulo with power-of-2
        ++head_;
    }

    int pop() {
        return data_[tail_++ & (N - 1)];
    }

    size_t size() const { return head_ - tail_; }
};

// Compile-time type invariants
template<typename T>
class TypeSafeContainer {
    static_assert(std::is_nothrow_move_constructible_v<T>,
        "T must be nothrow-moveable for exception safety");
    static_assert(!std::is_reference_v<T>,
        "Container cannot hold references");
    static_assert(sizeof(T) <= 1024,
        "T too large for inline storage");

    T value_;
public:
    explicit TypeSafeContainer(T val) : value_(std::move(val)) {}
    const T& get() const { return value_; }
};

int main() {
    RingBuffer<64> buf;  // OK: 64 is power of 2
    // RingBuffer<100> bad;  // COMPILE ERROR: not power of 2

    buf.push(42);
    std::cout << buf.pop() << "\n";

    TypeSafeContainer<int> c(42);
    // TypeSafeContainer<int&> bad;  // COMPILE ERROR
    return 0;
}

```

---

## Notes

- **Assertions for programmer errors** (internal bugs), **exceptions for runtime errors** (bad input)
- `assert()` is compiled out in release builds (`-DNDEBUG`) — zero cost in production
- Check invariants at the **end of every public mutating method**
- `static_assert` catches invariant violations at **compile time** — use it wherever possible
- C++26 contracts (`pre`, `post`, `assert`) will formalize this — start thinking in terms of contracts now
- Invariant checks during development catch bugs early; remove/disable in production for performance
