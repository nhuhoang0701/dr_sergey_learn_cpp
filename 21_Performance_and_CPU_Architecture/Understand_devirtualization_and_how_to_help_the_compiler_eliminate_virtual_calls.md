# Understand devirtualization and how to help the compiler eliminate virtual calls

**Category:** Performance & CPU Architecture  
**Item:** #719  
**Reference:** <https://en.cppreference.com/w/cpp/language/final>  

---

## Topic Overview

Devirtualization replaces indirect virtual calls (vtable lookup) with direct calls or inlined code, avoiding the ~5ns overhead per call.

| Devirt technique | When it works | Compiler flag |
| --- | --- | --- |
| Local type visible | `Derived d; d.func()` | `-O2` (default) |
| `final` class | Class marked `final` | `-O2` (default) |
| `final` method | Method marked `final` | `-O2` (default) |
| LTO whole-program | Only one derived class exists | `-flto` |
| PGO speculative | Profile shows one dominant type | `-fprofile-use` |

```asm

Virtual call (indirect):             Devirtualized (direct):
  mov rax, [rdi]       ; load vptr     call Derived::func  ; direct call
  mov rax, [rax + 16]  ; load func ptr (or inlined entirely)
  call rax             ; indirect call
  ~5ns overhead         ~0ns overhead

```

---

## Self-Assessment

### Q1: Devirtualization when dynamic type is locally visible

```cpp

#include <iostream>

struct Base {
    virtual int compute(int x) const { return x; }
    virtual ~Base() = default;
};

struct Derived : Base {
    int compute(int x) const override { return x * 2; }
};

struct Derived2 : Base {
    int compute(int x) const override { return x * 3; }
};

int main() {
    // CASE 1: Local object - compiler KNOWS the type
    Derived d;
    Base& ref = d;
    std::cout << ref.compute(5) << '\n';  // 10
    // Compiler sees: ref is actually Derived
    // Assembly: call Derived::compute  (direct, possibly inlined!)
    // No vtable lookup needed.

    // CASE 2: Stack-allocated, type visible
    Derived2 d2;
    Base* ptr = &d2;
    std::cout << ptr->compute(5) << '\n';  // 15
    // Compiler sees ptr = &d2, type is Derived2
    // Assembly: direct call or inlined

    // CASE 3: Dynamic allocation - compiler CAN devirtualize
    auto p = std::make_unique<Derived>();
    std::cout << p->compute(5) << '\n';  // 10
    // Compiler may still devirtualize: sees new Derived() in same scope

    // CASE 4: Factory function from another TU - compiler CANNOT devirtualize
    // Base* unknown = create_something();  // type unknown!
    // unknown->compute(5);  // must use vtable lookup

    // Verify with Compiler Explorer (godbolt.org):
    //   g++ -O2: Cases 1-3 show direct call, Case 4 shows indirect
}

```

### Q2: `final` enables devirtualization

```cpp

#include <iostream>
#include <memory>
#include <chrono>
#include <vector>

struct Shape {
    virtual double area() const = 0;
    virtual ~Shape() = default;
};

// Without final: compiler cannot devirtualize through pointers
struct CircleNoFinal : Shape {
    double radius;
    CircleNoFinal(double r) : radius(r) {}
    double area() const override { return 3.14159265 * radius * radius; }
};

// With final: compiler KNOWS no further derived classes exist
struct CircleFinal final : Shape {
    double radius;
    CircleFinal(double r) : radius(r) {}
    double area() const override { return 3.14159265 * radius * radius; }
};

// Can also mark individual methods as final:
struct Square : Shape {
    double side;
    Square(double s) : side(s) {}
    double area() const final { return side * side; }  // no override of area() possible
};

template<typename T>
[[gnu::noinline]] double sum_areas(const std::vector<T*>& shapes) {
    double total = 0;
    for (auto* s : shapes) {
        total += s->area();  // devirtualized if T is final!
    }
    return total;
}

int main() {
    constexpr int N = 1'000'000;
    std::vector<CircleNoFinal*> no_final(N);
    std::vector<CircleFinal*> with_final(N);

    for (int i = 0; i < N; ++i) {
        no_final[i] = new CircleNoFinal(1.0);
        with_final[i] = new CircleFinal(1.0);
    }

    auto bench = [](auto fn, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        volatile double r = 0;
        for (int i = 0; i < 100; ++i) r = fn();
        auto t1 = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    bench([&]{ return sum_areas(no_final); },   "Without final");
    bench([&]{ return sum_areas(with_final); }, "With final   ");
    // With final: direct call, possibly inlined -> 2-3x faster in tight loop

    for (auto* p : no_final) delete p;
    for (auto* p : with_final) delete p;
}

```

### Q3: Profile-Guided Optimization (PGO) speculative devirtualization

```cpp

#include <iostream>
#include <memory>
#include <vector>

struct Logger {
    virtual void log(const char* msg) = 0;
    virtual ~Logger() = default;
};

struct ConsoleLogger : Logger {
    void log(const char* msg) override {
        // In production: std::cout << msg << '\n';
    }
};

struct FileLogger : Logger {
    void log(const char* msg) override {
        // In production: write to file
    }
};

void process(Logger* logger, int n) {
    for (int i = 0; i < n; ++i) {
        logger->log("tick");  // virtual call
    }
}

// PGO speculative devirtualization works as follows:
//
// Step 1: Instrument build
//   g++ -O2 -fprofile-generate -o app app.cpp
//   ./app   # run with representative workload
//   # produces .gcda profile data files
//
// Step 2: Optimized build using profile
//   g++ -O2 -fprofile-use -o app app.cpp
//
// What the compiler does with profile data:
//   Profile shows: logger->log() calls ConsoleLogger::log 95% of the time.
//
//   Generated code becomes:
//     if (vtable_of(logger) == vtable_of(ConsoleLogger)) {
//         ConsoleLogger::log(logger, msg);  // DIRECT call (inlinable!)
//     } else {
//         logger->log(msg);  // fallback: indirect virtual call
//     }
//
// The speculative guard (type check) is very cheap (~1 cycle).
// For the 95% common case: direct call + possible inlining.
// For the 5% rare case: normal virtual call (no worse than before).
//
// Clang: -fprofile-instr-generate / -fprofile-instr-use
// MSVC:  /GL + /LTCG with profile-guided optimization

int main() {
    ConsoleLogger console;
    process(&console, 1000000);  // PGO profile captures this

    std::cout << "PGO speculative devirt:\n";
    std::cout << "  1. Profile: collect vtable usage statistics\n";
    std::cout << "  2. Compile: insert type guard for dominant type\n";
    std::cout << "  3. Runtime: dominant path is direct call\n";
}

```

---

## Notes

- `final` is the easiest devirtualization technique — zero runtime cost, just a type annotation.
- `-Rpass=devirt` (Clang) or `-fdump-ipa-devirt` (GCC) shows which calls were devirtualized.
- CRTP (Curiously Recurring Template Pattern) avoids virtual dispatch entirely at compile time.
- In tight loops, virtual call overhead can be 5-10x; devirtualization + inlining removes it.
- LTO (`-flto`) enables cross-TU devirtualization for the whole-program case.
