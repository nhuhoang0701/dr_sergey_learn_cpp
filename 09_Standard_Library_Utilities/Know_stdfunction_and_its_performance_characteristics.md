# Know std::function and its performance characteristics

**Category:** Standard Library — Utilities  
**Item:** #82  
**Standard:** C++11 / C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/function>  

---

## Topic Overview

`std::function<R(Args...)>` is a general-purpose polymorphic function wrapper that can store, copy, and invoke any callable target — lambdas, function pointers, functors, member function pointers (via `std::bind` or lambda).

### How It Works Internally

```cpp

std::function<int(int)> f = [x](int y){ return x + y; };

┌──────────────────────────────┐
│ std::function object         │
│  ┌────────────────────────┐  │
│  │ Small Buffer (SBO)     │  │  ← small callables stored inline
│  │ (typically 16-32 bytes)│  │
│  └────────────────────────┘  │
│  vtable* ──► invoke/copy/    │  ← type-erased call interface
│              destroy         │
│  ┌────────────────────────┐  │
│  │ heap ptr ──► callable  │  │  ← large callables heap-allocated
│  └────────────────────────┘  │
└──────────────────────────────┘

```

### Small Buffer Optimization (SBO)

- Implementations embed a small buffer (typically 16–32 bytes, implementation-defined).
- If the callable fits (size + alignment), it is stored inline — **no heap allocation**.
- If it doesn't fit, `std::function` allocates on the heap.

| Implementation | SBO buffer size |
| --- | --- |
| libstdc++ (GCC) | 16 bytes |
| libc++ (Clang) | 24 bytes |
| MSVC STL | 48 bytes (debug), variable |

### Performance Comparison

| Mechanism | Inline? | Heap alloc? | Virtual dispatch? | Copy? |
| --- | --- | --- | --- | --- |
| Template parameter | Yes | No | No | Yes |
| Function pointer | Often | No | No (direct call) | Yes |
| `std::function` (SBO) | Sometimes | No | Yes (type erasure) | Yes |
| `std::function` (heap) | No | **Yes** | Yes | Yes |
| `std::move_only_function` | Sometimes | No | Yes | **No** |

### Core Syntax

```cpp

#include <functional>
#include <iostream>

void free_func(int x) { std::cout << "free: " << x << "\n"; }

struct Functor {
    int state = 10;
    void operator()(int x) const { std::cout << "functor: " << x + state << "\n"; }
};

int main() {
    // Store a lambda
    std::function<int(int)> f = [](int x) { return x * 2; };
    std::cout << f(5) << "\n";  // Output: 10

    // Store a free function
    std::function<void(int)> g = free_func;
    g(42);  // Output: free: 42

    // Store a functor
    std::function<void(int)> h = Functor{20};
    h(5);   // Output: functor: 25

    // Check if empty
    std::function<void()> empty;
    if (!empty) std::cout << "empty!\n";  // Output: empty!
    // Calling an empty function throws std::bad_function_call
}

```

---

## Self-Assessment

### Q1: Explain the heap allocation that std::function performs for large callables and the small buffer optimization

**Answer:**

```cpp

#include <functional>
#include <iostream>
#include <array>

int main() {
    // --- Small callable: fits in SBO, no heap allocation ---
    int x = 42;
    std::function<int()> small = [x]{ return x; };
    // Captures one int (4-8 bytes) — fits in SBO buffer
    std::cout << small() << "\n";  // Output: 42

    // --- Large callable: exceeds SBO, triggers heap allocation ---
    std::array<char, 256> big_data{};
    big_data[0] = 'A';
    std::function<char()> large = [big_data]{ return big_data[0]; };
    // Captures 256 bytes — exceeds any SBO buffer → heap allocated
    std::cout << large() << "\n";  // Output: A

    // --- Demonstrate size ---
    std::cout << "sizeof(std::function<int()>) = "
              << sizeof(std::function<int()>) << "\n";
    // Typical output: 32 (libstdc++), 48 (libc++), 64 (MSVC)
    // This is the fixed size regardless of what callable it wraps

    // --- Proof: copying a large-callable function triggers another allocation ---
    auto copy = large;  // deep copy of the heap-allocated callable
    std::cout << copy() << "\n";  // Output: A
}

```

**How the SBO works:**

1. `std::function` has an internal buffer (e.g., 16 bytes on GCC).
2. When you assign a callable, it checks: `sizeof(callable) <= SBO_size && alignof(callable) <= SBO_align && callable is nothrow move constructible`.
3. If all conditions hold → stored inline (no allocation, fast).
4. Otherwise → `operator new` allocates memory, the callable is placed there, and `std::function` stores a pointer.
5. Every **copy** of a heap-allocated `std::function` triggers another heap allocation.

---

### Q2: Benchmark std::function vs direct function pointer vs template parameter functor

**Answer:**

```cpp

#include <functional>
#include <chrono>
#include <iostream>

// A simple operation to benchmark
inline int square(int x) { return x * x; }

struct SquareFunctor {
    int operator()(int x) const { return x * x; }
};

// Template parameter: compiler can inline the call
template<typename F>
long long sum_with_template(F f, int n) {
    long long total = 0;
    for (int i = 0; i < n; ++i) total += f(i);
    return total;
}

// Function pointer: indirect call, but no type erasure overhead
long long sum_with_fptr(int(*f)(int), int n) {
    long long total = 0;
    for (int i = 0; i < n; ++i) total += f(i);
    return total;
}

// std::function: type erasure + possible heap + virtual dispatch
long long sum_with_stdfunc(std::function<int(int)> f, int n) {
    long long total = 0;
    for (int i = 0; i < n; ++i) total += f(i);
    return total;
}

int main() {
    constexpr int N = 100'000'000;

    auto bench = [&](const char* label, auto callable) {
        auto start = std::chrono::high_resolution_clock::now();
        volatile long long result = callable();
        auto end = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
        std::cout << label << ": " << ms << " ms (result=" << result << ")\n";
    };

    bench("template<F>  ", [&]{ return sum_with_template(SquareFunctor{}, N); });
    bench("function ptr ", [&]{ return sum_with_fptr(square, N); });
    bench("std::function", [&]{ return sum_with_stdfunc(square, N); });

    // Typical results (GCC -O2):
    //   template<F>  :  45 ms  (inlined — fastest)
    //   function ptr :  90 ms  (indirect call)
    //   std::function: 150 ms  (type erasure overhead)
}

```

**Takeaway:** Templates allow full inlining and are the fastest. Function pointers add an indirect call. `std::function` adds virtual dispatch through type erasure. Use `std::function` only when you need runtime polymorphism for callables (e.g., callback registries, event systems).

---

### Q3: Show how std::move_only_function (C++23) solves the problem of non-copyable callable types

**Answer:**

```cpp

#include <functional>  // std::move_only_function (C++23)
#include <memory>
#include <iostream>
#include <vector>

int main() {
    // Problem: std::function requires the callable to be COPYABLE
    auto uptr = std::make_unique<int>(42);

    // This would NOT compile:
    // std::function<int()> f = [p = std::move(uptr)]{ return *p; };
    // Error: lambda capturing unique_ptr is not copyable

    // Solution: std::move_only_function (C++23)
    auto uptr2 = std::make_unique<int>(42);
    std::move_only_function<int()> f = [p = std::move(uptr2)]{ return *p; };
    std::cout << f() << "\n";  // Output: 42

    // move_only_function can be moved but NOT copied
    auto f2 = std::move(f);
    std::cout << f2() << "\n";  // Output: 42
    // f is now empty — calling it would be UB (no std::bad_function_call!)

    // Use case: task queue with move-only tasks
    std::vector<std::move_only_function<void()>> tasks;
    tasks.push_back([p = std::make_unique<std::string>("hello")]{
        std::cout << *p << "\n";
    });
    tasks.push_back([p = std::make_unique<std::string>("world")]{
        std::cout << *p << "\n";
    });

    for (auto& task : tasks) {
        task();
    }
    // Output:
    // hello
    // world
}

```

**Key differences from `std::function`:**

| Feature | `std::function` | `std::move_only_function` |
| --- | --- | --- |
| Copyable callable required | Yes | No |
| Copyable itself | Yes | No (move only) |
| `const` qualifier | Always calls as `const` | Supports `const`/non-`const`/`noexcept` qualifiers |
| Empty call behavior | Throws `bad_function_call` | Undefined behavior |
| Typical use | Callbacks, event handlers | Task queues, one-shot continuations |

---

## Notes

- `std::function` should be avoided in hot loops — prefer templates or `auto` parameters.
- In C++20, you can use `auto` or concepts for callable parameters: `void foo(std::invocable<int> auto f)`.
- `std::function` is **not** suitable for storing member function pointers directly — wrap them with a lambda or `std::bind`.
- `std::move_only_function` allows specifying CV qualifiers and `noexcept` on the signature: `std::move_only_function<int(int) const noexcept>`.
- If you only need a non-owning callable reference, use `std::function_ref` (C++26) or a template parameter.

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
