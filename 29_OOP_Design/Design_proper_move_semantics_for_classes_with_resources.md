# Design proper move semantics for classes with resources

**Category:** OOP Design

---

## Topic Overview

Move semantics transfer ownership of resources from one object to another **without copying**:

```cpp

Copy:  old object KEEPS resource, new object gets a DUPLICATE
Move:  old object LOSES resource, new object TAKES it (cheap!)

```

| Operation | What Happens | Cost | Source After |
| --- | --- | --- | --- |
| Copy construct | Deep copy | O(n) | Unchanged |
| Move construct | Pointer swap | **O(1)** | Valid but empty |
| Copy assign | Free old + deep copy | O(n) | Unchanged |
| Move assign | Free old + pointer swap | **O(1)** | Valid but empty |

---

## Self-Assessment

### Q1: Implement a resource-owning class with correct move semantics

**Answer:**

```cpp

#include <cstring>
#include <utility>
#include <algorithm>
#include <iostream>

class DynamicArray {
    double* data_;
    size_t size_;

public:
    // Constructor: acquires resource
    explicit DynamicArray(size_t n)
        : data_(n ? new double[n]{} : nullptr), size_(n) {}

    // Destructor: releases resource
    ~DynamicArray() { delete[] data_; }

    // Copy constructor: deep copy
    DynamicArray(const DynamicArray& other)
        : data_(other.size_ ? new double[other.size_] : nullptr)
        , size_(other.size_) {
        std::memcpy(data_, other.data_, size_ * sizeof(double));
    }

    // Move constructor: steal resource (MUST be noexcept!)
    DynamicArray(DynamicArray&& other) noexcept
        : data_(std::exchange(other.data_, nullptr))
        , size_(std::exchange(other.size_, 0)) {}

    // Copy assignment: copy-and-swap idiom
    DynamicArray& operator=(DynamicArray other) noexcept {
        swap(*this, other);  // other dies with old data
        return *this;
    }
    // Handles both copy and move assignment!
    // - copy arg: other is copy-constructed (deep copy)
    // - move arg: other is move-constructed (cheap)

    friend void swap(DynamicArray& a, DynamicArray& b) noexcept {
        using std::swap;
        swap(a.data_, b.data_);
        swap(a.size_, b.size_);
    }

    double& operator[](size_t i) { return data_[i]; }
    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
};

int main() {
    DynamicArray a(1000000);  // 1M doubles
    a[0] = 3.14;

    DynamicArray b = std::move(a);  // O(1)! Just pointer swap
    std::cout << "a.size=" << a.size()   // 0 (moved-from)
              << " b.size=" << b.size()  // 1000000
              << " b[0]=" << b[0] << "\n";  // 3.14
    return 0;
}

```

### Q2: Handle classes with multiple resources correctly

**Answer:**

```cpp

#include <memory>
#include <string>
#include <vector>
#include <utility>
#include <iostream>

// When a class holds MULTIPLE resources, use RAII members
// so the Rule of Zero applies
class Connection {
    std::unique_ptr<char[]> buffer_;   // Resource 1
    std::string name_;                  // Resource 2 (self-managing)
    std::vector<int> pending_;          // Resource 3 (self-managing)
    int socket_fd_;                     // Resource 4 (raw handle)

public:
    Connection(std::string name, size_t buf_size)
        : buffer_(std::make_unique<char[]>(buf_size))
        , name_(std::move(name))
        , socket_fd_(-1) {}

    // Rule of Zero: compiler-generated move is correct!
    // unique_ptr, string, vector all know how to move themselves.
    Connection(Connection&&) = default;
    Connection& operator=(Connection&&) = default;

    // BUT: we need custom destructor for socket_fd_
    ~Connection() {
        if (socket_fd_ >= 0) {
            // close(socket_fd_);  // POSIX close
            std::cout << "Closed socket " << socket_fd_ << "\n";
        }
    }

    // Since we have custom destructor, explicitly default move
    // (Rule of Five: if you define one, define them all)
    // Copy doesn't make sense for connections
    Connection(const Connection&) = delete;
    Connection& operator=(const Connection&) = delete;

    const std::string& name() const { return name_; }
};

// BETTER: wrap raw handle in RAII, then Rule of Zero
class SocketHandle {
    int fd_;
public:
    explicit SocketHandle(int fd = -1) : fd_(fd) {}
    ~SocketHandle() { if (fd_ >= 0) { /* close */ } }
    SocketHandle(SocketHandle&& o) noexcept : fd_(std::exchange(o.fd_, -1)) {}
    SocketHandle& operator=(SocketHandle&& o) noexcept {
        if (this != &o) {
            if (fd_ >= 0) { /* close */ }
            fd_ = std::exchange(o.fd_, -1);
        }
        return *this;
    }
    SocketHandle(const SocketHandle&) = delete;
    int get() const { return fd_; }
};

class BetterConnection {
    std::unique_ptr<char[]> buffer_;
    std::string name_;
    SocketHandle socket_;  // RAII handle
    // Rule of Zero: ALL members are self-managing!
};

```

### Q3: Show moves in containers and the importance of noexcept

**Answer:**

```cpp

#include <vector>
#include <string>
#include <iostream>
#include <utility>

class Widget {
    std::string name_;
    int* data_;
    size_t size_;
    static int move_count;
    static int copy_count;

public:
    Widget(std::string name, size_t n)
        : name_(std::move(name)), data_(new int[n]{}), size_(n) {}

    ~Widget() { delete[] data_; }

    // NOEXCEPT move = vector uses move during reallocation
    Widget(Widget&& o) noexcept
        : name_(std::move(o.name_))
        , data_(std::exchange(o.data_, nullptr))
        , size_(std::exchange(o.size_, 0)) {
        ++move_count;
    }

    Widget(const Widget& o)
        : name_(o.name_), data_(new int[o.size_]), size_(o.size_) {
        std::copy(o.data_, o.data_ + size_, data_);
        ++copy_count;
    }

    Widget& operator=(Widget o) noexcept {
        std::swap(name_, o.name_);
        std::swap(data_, o.data_);
        std::swap(size_, o.size_);
        return *this;
    }

    static void print_stats() {
        std::cout << "Moves: " << move_count << ", Copies: " << copy_count << "\n";
    }
};

int Widget::move_count = 0;
int Widget::copy_count = 0;

int main() {
    std::vector<Widget> v;
    // Reserve to see the effect of noexcept
    for (int i = 0; i < 100; ++i)
        v.emplace_back("Widget" + std::to_string(i), 1000);

    Widget::print_stats();
    // With noexcept move: "Moves: N, Copies: 0"
    // Without noexcept:  "Moves: 0, Copies: N" (vector falls back to copy!)
    return 0;
}

```

---

## Notes

- **Always mark move operations `noexcept`** — `std::vector` only uses move if it's guaranteed not to throw
- Use `std::exchange(member, sentinel)` in move constructors for clean, exception-safe transfer
- Moved-from objects must be in a "valid but unspecified" state — at minimum, destructible and assignable
- Prefer the **copy-and-swap** idiom for unified copy+move assignment
- Wrap raw resources in RAII types, then the containing class follows the Rule of Zero
- `static_assert(std::is_nothrow_move_constructible_v<T>)` catches missing `noexcept` at compile time
