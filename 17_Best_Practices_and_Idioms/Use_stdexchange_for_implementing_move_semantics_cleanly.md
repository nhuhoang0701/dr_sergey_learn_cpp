# Use std::exchange for implementing move semantics cleanly

**Category:** Best Practices & Idioms  
**Item:** #145  
**Reference:** <https://en.cppreference.com/w/cpp/utility/exchange>  

---

## Topic Overview

`std::exchange(obj, new_val)` replaces `obj` with `new_val` and returns the **old** value of `obj` — all in one expression. This makes move constructors and move assignment operators much cleaner.

```cpp

#include <utility>
int old = std::exchange(x, 0);  // old = x, then x = 0

```

---

## Self-Assessment

### Q1: Implement a move constructor using `std::exchange`

```cpp

#include <iostream>
#include <utility>
#include <cstddef>

class Buffer {
    int* data_;
    size_t size_;
public:
    explicit Buffer(size_t n)
        : data_(new int[n]{}), size_(n) {}

    ~Buffer() { delete[] data_; }

    // Move constructor — clean with std::exchange
    Buffer(Buffer&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))  // take pointer, set source to null
        , size_(std::exchange(other.size_, 0))         // take size, set source to 0
    {}

    // Move assignment
    Buffer& operator=(Buffer&& other) noexcept {
        if (this != &other) {
            delete[] data_;
            data_ = std::exchange(other.data_, nullptr);
            size_ = std::exchange(other.size_, 0);
        }
        return *this;
    }

    // Delete copy
    Buffer(const Buffer&) = delete;
    Buffer& operator=(const Buffer&) = delete;

    size_t size() const { return size_; }
    bool empty() const { return data_ == nullptr; }
};

int main() {
    Buffer a(100);
    std::cout << "a.size = " << a.size() << ", empty = " << a.empty() << '\n';

    Buffer b(std::move(a));
    std::cout << "After move:\n";
    std::cout << "a.size = " << a.size() << ", empty = " << a.empty() << '\n';
    std::cout << "b.size = " << b.size() << ", empty = " << b.empty() << '\n';
}
// Expected output:
// a.size = 100, empty = 0
// After move:
// a.size = 0, empty = 1
// b.size = 100, empty = 0

```

### Q2: Show that `std::exchange` returns the old value

```cpp

#include <iostream>
#include <string>
#include <utility>
#include <list>

int main() {
    // Basic usage: returns old, assigns new
    int x = 42;
    int old = std::exchange(x, 0);
    std::cout << "old=" << old << " x=" << x << '\n';  // old=42 x=0

    // With strings
    std::string name = "Alice";
    std::string prev = std::exchange(name, "Bob");
    std::cout << "prev=" << prev << " name=" << name << '\n';  // prev=Alice name=Bob

    // Cycling through values (rotate)
    int a = 1, b = 2, c = 3;
    a = std::exchange(b, std::exchange(c, a));  // rotate a->c->b->a
    std::cout << a << " " << b << " " << c << '\n';  // 2 3 1

    // Linked list traversal
    struct Node { int val; Node* next; };
    Node n3{3, nullptr}, n2{2, &n3}, n1{1, &n2};
    Node* curr = &n1;
    while (curr) {
        std::cout << curr->val << ' ';
        curr = std::exchange(curr->next, nullptr);  // advance and disconnect
    }
    std::cout << '\n';
}
// Expected output:
// old=42 x=0
// prev=Alice name=Bob
// 2 3 1
// 1 2

```

### Q3: Compare `std::exchange` with manual temp variable pattern

```cpp

#include <iostream>
#include <utility>

class Handle {
    int fd_;
public:
    explicit Handle(int fd) : fd_(fd) {}
    ~Handle() { if (fd_ >= 0) std::cout << "close(" << fd_ << ")\n"; }

    // WITHOUT std::exchange (verbose, error-prone)
    Handle(Handle&& other) noexcept {
        fd_ = other.fd_;        // take
        other.fd_ = -1;         // reset source
        // Two separate steps — easy to forget the reset!
    }

    // WITH std::exchange (one-liner, atomic intent)
    // Handle(Handle&& other) noexcept
    //     : fd_(std::exchange(other.fd_, -1)) {}  // take + reset in one expression

    Handle& operator=(Handle&& other) noexcept {
        if (this != &other)
            fd_ = std::exchange(other.fd_, -1);
        return *this;
    }

    Handle(const Handle&) = delete;
    Handle& operator=(const Handle&) = delete;

    int get() const { return fd_; }
};

int main() {
    Handle h1(42);
    Handle h2(std::move(h1));
    std::cout << "h1=" << h1.get() << " h2=" << h2.get() << '\n';
}
// Expected output:
// h1=-1 h2=42
// close(42)

```

| Pattern | Lines | Clarity | Bug risk |
| --- | --- | --- | --- |
| `tmp = x; x = new; use tmp;` | 3 | Low | Forget reset |
| `std::exchange(x, new)` | 1 | High | Minimal |

---

## Notes

- `std::exchange` is in `<utility>` since C++14.
- It's `constexpr` since C++20.
- Use it in move constructors, move assignment, linked-list iteration, state machines.
- It does NOT do an atomic exchange — use `std::atomic::exchange` for that.
