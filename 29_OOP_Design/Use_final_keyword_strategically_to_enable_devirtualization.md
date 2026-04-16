# Use final keyword strategically to enable devirtualization

**Category:** OOP Design

---

## Topic Overview

The `final` specifier tells the compiler that a class or virtual function cannot be overridden further, enabling **devirtualization** — replacing indirect vtable calls with direct calls or inlining:

```cpp

Without final:   obj->draw()  →  vtable lookup →  indirect call (branch misprediction)
With final:      obj->draw()  →  direct call →  possibly inlined (zero overhead)

```

| Usage | Syntax | Effect |
| --- | --- | --- |
| Final class | `class Leaf final : public Base {}` | No further derivation |
| Final method | `void draw() const final {}` | No further override |
| Both | Class final implies all methods final | Maximum optimization |

---

## Self-Assessment

### Q1: Show devirtualization with and without final

**Answer:**

```cpp

#include <memory>
#include <vector>
#include <numeric>
#include <iostream>
#include <chrono>

class Shape {
public:
    virtual ~Shape() = default;
    virtual double area() const = 0;
};

// Without final: compiler can't devirtualize
class CircleOpen : public Shape {
    double r_;
public:
    explicit CircleOpen(double r) : r_(r) {}
    double area() const override { return 3.14159265 * r_ * r_; }
};

// With final: compiler KNOWS no further overrides exist
class CircleFinal final : public Shape {
    double r_;
public:
    explicit CircleFinal(double r) : r_(r) {}
    double area() const override { return 3.14159265 * r_ * r_; }
};

// When the compiler can prove the dynamic type, it devirtualizes
template<typename T>
double sum_areas(const std::vector<T>& shapes, size_t n) {
    double total = 0;
    for (size_t i = 0; i < n; ++i)
        total += shapes[i].area();  // Devirtualized if T is final!
    return total;
}

int main() {
    // Devirtualization happens with concrete final type
    std::vector<CircleFinal> finals(10000, CircleFinal(1.0));
    auto result = sum_areas(finals, finals.size());  // Direct call, possibly inlined
    std::cout << "Sum: " << result << "\n";

    // Through pointer: compiler may still devirtualize with final
    auto p = std::make_unique<CircleFinal>(5.0);
    std::cout << p->area() << "\n";  // Compiler knows it's CircleFinal
    return 0;
}

```

### Q2: Use final to seal a class hierarchy at the right level

**Answer:**

```cpp

#include <iostream>

// Open for extension
class Logger {
public:
    virtual ~Logger() = default;
    virtual void log(const char* msg) = 0;
    virtual void flush() { /* default: no-op */ }
};

// Intermediate: can be extended further
class FileLogger : public Logger {
protected:
    void write_to_file(const char* msg) {
        std::cout << "[FILE] " << msg << "\n";
    }
public:
    void log(const char* msg) override {
        write_to_file(msg);
    }
};

// FINAL: this is the leaf — no further extension
class RotatingFileLogger final : public FileLogger {
    int max_lines_ = 1000;
    int current_line_ = 0;
public:
    void log(const char* msg) override {
        if (++current_line_ > max_lines_) {
            // rotate logic
            current_line_ = 0;
        }
        write_to_file(msg);
    }
    void flush() override { std::cout << "[FLUSH]\n"; }
};

// Final method without final class
class BufferedLogger : public Logger {
public:
    // Subclasses MUST NOT override log — but CAN override flush
    void log(const char* msg) final {
        buffer_ += msg;
        buffer_ += '\n';
        if (buffer_.size() > 4096) flush();
    }
    void flush() override {
        std::cout << buffer_;
        buffer_.clear();
    }
private:
    std::string buffer_;
};

int main() {
    RotatingFileLogger logger;
    logger.log("Hello");
    logger.flush();
    return 0;
}

```

### Q3: Demonstrate the performance impact with a benchmark

**Answer:**

```cpp

#include <vector>
#include <memory>
#include <chrono>
#include <iostream>
#include <cmath>

class Base {
public:
    virtual ~Base() = default;
    virtual double compute(double x) const = 0;
};

class OpenImpl : public Base {
public:
    double compute(double x) const override {
        return std::sin(x) * std::cos(x);
    }
};

class FinalImpl final : public Base {
public:
    double compute(double x) const override {
        return std::sin(x) * std::cos(x);
    }
};

template<typename T>
double bench_concrete(const T& obj, int iterations) {
    double sum = 0;
    for (int i = 0; i < iterations; ++i)
        sum += obj.compute(static_cast<double>(i));  // Devirtualized!
    return sum;
}

double bench_virtual(const Base& obj, int iterations) {
    double sum = 0;
    for (int i = 0; i < iterations; ++i)
        sum += obj.compute(static_cast<double>(i));  // Virtual call
    return sum;
}

int main() {
    constexpr int N = 10'000'000;
    FinalImpl fi;
    OpenImpl oi;

    auto t1 = std::chrono::high_resolution_clock::now();
    volatile double r1 = bench_concrete(fi, N);  // Template: devirtualized
    auto t2 = std::chrono::high_resolution_clock::now();
    volatile double r2 = bench_virtual(oi, N);    // Virtual dispatch
    auto t3 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::milliseconds;
    std::cout << "Devirtualized: " << std::chrono::duration_cast<ms>(t2-t1).count() << "ms\n";
    std::cout << "Virtual:       " << std::chrono::duration_cast<ms>(t3-t2).count() << "ms\n";
    return 0;
}

```

---

## Notes

- `final` enables **devirtualization**: the compiler replaces vtable lookups with direct/inlined calls
- Mark leaf classes `final` by default — you can always remove it later if extension is needed
- `final` on a method prevents override in subclasses without preventing class extension
- GCC/Clang with `-O2` are aggressive about devirtualization when final types are visible
- LTO (Link-Time Optimization) can devirtualize even without `final` if it can prove the type
- Performance impact: significant in tight loops (eliminates branch misprediction + enables inlining)
