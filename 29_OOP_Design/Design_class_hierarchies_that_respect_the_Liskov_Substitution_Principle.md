# Design class hierarchies that respect the Liskov Substitution Principle

**Category:** OOP Design

---

## Topic Overview

**Liskov Substitution Principle (LSP):** If `S` is a subtype of `T`, then objects of type `T` can be replaced with objects of type `S` without altering the correctness of the program. In C++ terms: every `Derived*` must work correctly wherever a `Base*` is expected.

### LSP Violation Checklist

| Violation | Example | Fix |
| --- | --- | --- |
| Strengthening preconditions | Derived rejects inputs Base accepts | Weaken or keep same preconditions |
| Weakening postconditions | Derived returns less than promised | Keep or strengthen postconditions |
| Throwing unexpected exceptions | Derived throws where Base doesn't | Only throw subtypes of Base's exceptions |
| Mutating invariants | `Square::setWidth()` also sets height | Don't inherit — separate types |
| Removing functionality | Derived's `save()` is a no-op | Don't inherit if you can't fulfill contract |

---

## Self-Assessment

### Q1: Show a classic LSP violation and how to fix it

**Answer:**

```cpp

#include <stdexcept>
#include <iostream>
#include <memory>
#include <vector>
#include <cassert>

// ═══════════ VIOLATION: Rectangle/Square problem ═══════════
class Rectangle {
protected:
    int w_, h_;
public:
    Rectangle(int w, int h) : w_(w), h_(h) {}
    virtual void set_width(int w)  { w_ = w; }
    virtual void set_height(int h) { h_ = h; }
    int area() const { return w_ * h_; }
};

class Square : public Rectangle {  // LSP VIOLATION!
public:
    explicit Square(int side) : Rectangle(side, side) {}
    void set_width(int w) override  { w_ = h_ = w; }  // Side effect!
    void set_height(int h) override { w_ = h_ = h; }  // Side effect!
};

void test_rectangle(Rectangle& r) {
    r.set_width(5);
    r.set_height(3);
    assert(r.area() == 15);  // FAILS for Square! (area == 9)
    // Client rightfully expects independent width/height
}

// ═══════════ FIX 1: Immutable value types — no setters, no problem ═══════════
struct Rect {
    int width, height;
    int area() const { return width * height; }
    Rect with_width(int w) const { return {w, height}; }
    Rect with_height(int h) const { return {width, h}; }
};

struct Sqr {
    int side;
    int area() const { return side * side; }
};
// No inheritance relationship — no LSP issue

// ═══════════ FIX 2: Interface that both can fulfill ═══════════
class Shape {
public:
    virtual ~Shape() = default;
    virtual int area() const = 0;  // read-only interface — no setters to violate
};

class RectShape : public Shape {
    int w_, h_;
public:
    RectShape(int w, int h) : w_(w), h_(h) {}
    int area() const override { return w_ * h_; }
};

class SquareShape : public Shape {
    int side_;
public:
    explicit SquareShape(int s) : side_(s) {}
    int area() const override { return side_ * side_; }
};

// LSP holds: any Shape* can compute area() correctly
void print_area(const Shape& s) {
    std::cout << "Area: " << s.area() << "\n";  // Always correct
}

```

### Q2: Design a real-world hierarchy that respects LSP with contracts

**Answer:**

```cpp

#include <string>
#include <memory>
#include <vector>
#include <stdexcept>

// ═══════════ Transport system with documented contracts ═══════════
class Transport {
public:
    virtual ~Transport() = default;

    // CONTRACT:
    // Precondition:  passengers >= 0 && passengers <= capacity()
    // Postcondition: returns estimated minutes > 0
    // Invariant:     capacity() > 0
    virtual int estimate_travel_time(int distance_km) const = 0;

    // CONTRACT:
    // Postcondition: returns > 0
    virtual int capacity() const = 0;

    // CONTRACT:
    // Precondition:  passengers >= 0
    // Postcondition: returns cost >= 0.0
    virtual double calculate_fare(int passengers, int distance_km) const = 0;
};

class Bus : public Transport {
public:
    int estimate_travel_time(int distance_km) const override {
        return distance_km * 3;  // 20 km/h average with stops
    }
    int capacity() const override { return 50; }
    double calculate_fare(int passengers, int distance_km) const override {
        return passengers * distance_km * 0.10;  // $0.10/km/person
    }
};

class Taxi : public Transport {
public:
    int estimate_travel_time(int distance_km) const override {
        return distance_km * 2;  // 30 km/h average
    }
    int capacity() const override { return 4; }
    double calculate_fare(int passengers, int distance_km) const override {
        return 3.50 + distance_km * 1.80;  // Base fare + per km
    }
};

// LSP-SAFE derived class: overrides but maintains contracts
class ExpressBus : public Bus {
public:
    int estimate_travel_time(int distance_km) const override {
        return distance_km * 2;  // Faster (postcondition: still > 0 ✓)
    }
    // capacity() inherited = 50 (invariant: > 0 ✓)
    double calculate_fare(int passengers, int distance_km) const override {
        return Bus::calculate_fare(passengers, distance_km) * 1.5;  // More expensive (>= 0 ✓)
    }
};

// This works for ANY Transport — LSP guarantee
void plan_trip(const Transport& t, int passengers, int km) {
    if (passengers > t.capacity())
        throw std::invalid_argument("Too many passengers");
    int time = t.estimate_travel_time(km);
    double cost = t.calculate_fare(passengers, km);
    // time > 0 guaranteed by contract
    // cost >= 0 guaranteed by contract
}

// ═══════════ LSP VIOLATION — caught by contract ═══════════
class BrokenScooter : public Transport {
public:
    int estimate_travel_time(int distance_km) const override {
        if (distance_km > 50) return -1;  // VIOLATION! Must return > 0
        return distance_km;
    }
    int capacity() const override { return 1; }
    double calculate_fare(int, int) const override { return 0; }
    // Returns 0 always — while not technically violating >= 0,
    // clients depend on fare being meaningful. Contract need not be formal.
};

```

### Q3: Show an LSP-compliant design for a stream/IO hierarchy

**Answer:**

```cpp

#include <cstdint>
#include <span>
#include <vector>
#include <cstring>
#include <stdexcept>

// ═══════════ LSP-compliant stream hierarchy ═══════════
class InputStream {
public:
    virtual ~InputStream() = default;

    // CONTRACT:
    // Pre:  buffer.size() > 0
    // Post: returns bytes read, 0 <= result <= buffer.size()
    //       result == 0 means end of stream
    virtual size_t read(std::span<uint8_t> buffer) = 0;

    // Post: returns true if no more data available
    virtual bool eof() const = 0;

    // Helper (non-virtual, uses read())
    std::vector<uint8_t> read_all() {
        std::vector<uint8_t> result;
        uint8_t buf[4096];
        while (!eof()) {
            size_t n = read(buf);
            if (n == 0) break;
            result.insert(result.end(), buf, buf + n);
        }
        return result;
    }
};

class MemoryStream : public InputStream {
    std::span<const uint8_t> data_;
    size_t pos_ = 0;
public:
    explicit MemoryStream(std::span<const uint8_t> data) : data_(data) {}

    size_t read(std::span<uint8_t> buffer) override {
        size_t avail = data_.size() - pos_;
        size_t to_read = std::min(buffer.size(), avail);
        std::memcpy(buffer.data(), data_.data() + pos_, to_read);
        pos_ += to_read;
        return to_read;  // 0 <= result <= buffer.size() ✓
    }

    bool eof() const override { return pos_ >= data_.size(); }
};

// Decorator: adds buffering — LSP compliant (wraps another InputStream)
class BufferedStream : public InputStream {
    std::unique_ptr<InputStream> inner_;
    std::vector<uint8_t> buf_;
    size_t buf_pos_ = 0, buf_len_ = 0;

public:
    BufferedStream(std::unique_ptr<InputStream> inner, size_t buf_size = 8192)
        : inner_(std::move(inner)), buf_(buf_size) {}

    size_t read(std::span<uint8_t> buffer) override {
        if (buf_pos_ >= buf_len_) {
            buf_len_ = inner_->read(buf_);
            buf_pos_ = 0;
            if (buf_len_ == 0) return 0;
        }
        size_t avail = buf_len_ - buf_pos_;
        size_t to_copy = std::min(buffer.size(), avail);
        std::memcpy(buffer.data(), buf_.data() + buf_pos_, to_copy);
        buf_pos_ += to_copy;
        return to_copy;  // Fulfills same contract as MemoryStream ✓
    }

    bool eof() const override {
        return buf_pos_ >= buf_len_ && inner_->eof();
    }
};

// Client code doesn't care which stream type —  LSP guarantee
size_t count_bytes(InputStream& stream) {
    size_t total = 0;
    uint8_t buf[1024];
    while (auto n = stream.read(buf)) {
        total += n;
    }
    return total;
}

```

---

## Notes

- **LSP is about behavioral compatibility**, not just type compatibility. The compiler can't enforce it — you must design for it
- Rule of thumb: if a derived class needs to disable or throw from an inherited method, LSP is violated
- Prefer **narrow, read-only interfaces** — they're easier to satisfy across all implementations
- Immutable value types trivially satisfy LSP (no state mutation = no violated postconditions)
- **Design by Contract** (pre/post conditions, invariants) makes LSP violations visible and testable
