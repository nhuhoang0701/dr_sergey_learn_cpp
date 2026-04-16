# Understand Stack vs Heap Allocation Tradeoffs and Prefer Stack Allocation

**Category:** Memory & Ownership  
**Item:** #33  
**Reference:** <https://en.cppreference.com/w/cpp/memory>  

---

## Topic Overview

### Stack vs Heap Comparison

| Feature | Stack | Heap |
| --- | --- | --- |
| **Speed** | ~1 CPU instruction (adjust SP) | System call / allocator walk |
| **Deallocation** | Automatic (scope exit) | Manual (`delete`) or smart ptr |
| **Size limit** | Small (1-8 MB typical) | Limited only by system RAM |
| **Fragmentation** | None | Can fragment over time |
| **Cache locality** | **Excellent** (hot in L1) | Variable (scattered) |
| **Thread safety** | Inherently thread-local | Requires synchronization |
| **Lifetime control** | Scope-bound only | Arbitrary (cross-scope) |

### Rule of Thumb

**Prefer stack allocation** unless:

1. Object is too large for the stack
2. Object must outlive the current scope
3. Size is unknown at compile time
4. Object is polymorphic (dynamic type)

---

## Self-Assessment

### Q1: Benchmark a hot loop that allocates on heap vs stack and quantify the difference

```cpp

#include <iostream>
#include <chrono>
#include <memory>
#include <array>
#include <numeric>
#include <cstdlib>

struct Data {
    int values[64];  // 256 bytes
};

template<typename Func>
long long bench(Func f, int iterations) {
    auto start = std::chrono::high_resolution_clock::now();
    volatile int sink = 0;
    for (int i = 0; i < iterations; ++i) {
        sink += f();
    }
    auto end = std::chrono::high_resolution_clock::now();
    return std::chrono::duration_cast<std::chrono::microseconds>(end - start).count();
}

int main() {
    constexpr int N = 1'000'000;

    // Stack allocation: just adjust stack pointer
    auto stack_time = bench([]() -> int {
        Data d;
        d.values[0] = 42;
        return d.values[0];
    }, N);

    // Heap allocation: malloc + free per iteration
    auto heap_time = bench([]() -> int {
        Data* d = new Data;
        d->values[0] = 42;
        int result = d->values[0];
        delete d;
        return result;
    }, N);

    // Heap with unique_ptr
    auto unique_time = bench([]() -> int {
        auto d = std::make_unique<Data>();
        d->values[0] = 42;
        return d->values[0];
    }, N);

    std::cout << "=== " << N << " allocations of " << sizeof(Data) << " bytes ===\n";
    std::cout << "Stack:      " << stack_time << " us\n";
    std::cout << "Heap (raw): " << heap_time << " us\n";
    std::cout << "Heap (up):  " << unique_time << " us\n";
    std::cout << "Heap/Stack: " << (double)heap_time / (stack_time > 0 ? stack_time : 1) << "x slower\n";

    // Cache locality demonstration
    std::cout << "\n=== Cache locality ===\n";
    constexpr int ARR_SIZE = 10000;

    // Contiguous (stack-like)
    auto cache_good = bench([&]() -> int {
        int arr[ARR_SIZE];
        for (int i = 0; i < ARR_SIZE; ++i) arr[i] = i;
        return arr[ARR_SIZE / 2];
    }, 1000);

    // Scattered (heap pointers)
    auto cache_bad = bench([&]() -> int {
        std::vector<std::unique_ptr<int>> ptrs(ARR_SIZE);
        for (int i = 0; i < ARR_SIZE; ++i) ptrs[i] = std::make_unique<int>(i);
        return *ptrs[ARR_SIZE / 2];
    }, 1000);

    std::cout << "Contiguous array:    " << cache_good << " us\n";
    std::cout << "Scattered pointers:  " << cache_bad << " us\n";

    return 0;
}

```

### Q2: Explain why large local arrays can cause stack overflow and how to detect it

```cpp

#include <iostream>
#include <thread>
#include <cstddef>

void demonstrate_stack_limits() {
    std::cout << "=== Stack overflow risk ===\n\n";

    // Typical stack sizes:
    // Linux: 8 MB (ulimit -s)
    // Windows: 1 MB default
    // Threads: often 1-2 MB

    // This is FINE (4 KB on stack):
    int small_array[1024];  // 4 KB
    small_array[0] = 42;
    std::cout << "Small array (4 KB): OK\n";

    // This would CRASH on many platforms (~4 MB):
    // int huge_array[1000000];  // ~4 MB — may exceed stack!
    // huge_array[0] = 1;  // STACK OVERFLOW

    // Even worse in recursive functions:
    // void recurse(int depth) {
    //     int local[1000];  // 4KB per frame
    //     if (depth > 0) recurse(depth - 1);
    //     // 1000 recursions = 4MB of stack
    // }

    std::cout << "\nStack size guidelines:\n";
    std::cout << "  < 1 KB:    Always safe on stack\n";
    std::cout << "  1-64 KB:   Usually safe, be cautious in recursion\n";
    std::cout << "  > 64 KB:   Consider heap allocation\n";
    std::cout << "  > 1 MB:    MUST use heap (will overflow most stacks)\n";

    std::cout << "\nDetection methods:\n";
    std::cout << "  1. -fsanitize=address (ASan detects stack overflow)\n";
    std::cout << "  2. -Wframe-larger-than=N (compiler warning)\n";
    std::cout << "  3. ulimit -s (check/set stack size on Linux)\n";
    std::cout << "  4. Stack guard pages (OS-level protection)\n";
    std::cout << "  5. /STACK:size (MSVC linker option)\n";
}

int main() {
    demonstrate_stack_limits();

    // Thread stacks are often smaller
    std::cout << "\n=== Thread stack risk ===\n";
    std::thread t([] {
        // Thread stacks are typically 1-2 MB
        // Large locals here are even more dangerous
        int arr[100];  // OK
        arr[0] = 1;
        std::cout << "Thread stack usage: safe (small array)\n";
    });
    t.join();

    return 0;
}

```

### Q3: Demonstrate using `std::array` or a local `vector` with `reserve` as `alloca` alternatives

```cpp

#include <iostream>
#include <array>
#include <vector>
#include <numeric>
#include <chrono>

// alloca() allocates on the stack but:
// - Not standard C++
// - Size not checked (overflow = crash)
// - VLAs are similar (not in C++ standard)

// Better alternatives:

// Method 1: std::array — compile-time size, stack-allocated
template<size_t N>
void process_fixed(int seed) {
    std::array<int, N> data;
    std::iota(data.begin(), data.end(), seed);
    std::cout << "std::array<int, " << N << ">: "
              << "sizeof=" << sizeof(data) << ", "
              << "sum=" << std::accumulate(data.begin(), data.end(), 0LL) << "\n";
}

// Method 2: Vector with reserve — heap, but one allocation
void process_dynamic(size_t n, int seed) {
    std::vector<int> data;
    data.reserve(n);  // Single allocation, no reallocations
    for (size_t i = 0; i < n; ++i) {
        data.push_back(seed + static_cast<int>(i));
    }
    std::cout << "vector.reserve(" << n << "): "
              << "sum=" << std::accumulate(data.begin(), data.end(), 0LL) << "\n";
}

// Method 3: Small buffer optimization — try stack, fall back to heap
template<size_t StackSize = 256>
void process_sbo(size_t n, int seed) {
    // Stack buffer for small inputs
    std::array<int, StackSize> stack_buf;

    int* data;
    std::vector<int> heap_buf;  // Used only if n > StackSize

    if (n <= StackSize) {
        data = stack_buf.data();
        std::cout << "SBO: using STACK (" << n << " <= " << StackSize << ")\n";
    } else {
        heap_buf.resize(n);
        data = heap_buf.data();
        std::cout << "SBO: using HEAP (" << n << " > " << StackSize << ")\n";
    }

    for (size_t i = 0; i < n; ++i) data[i] = seed + static_cast<int>(i);

    long long sum = 0;
    for (size_t i = 0; i < n; ++i) sum += data[i];
    std::cout << "  sum=" << sum << "\n";
}

int main() {
    std::cout << "=== Method 1: std::array (compile-time size) ===\n";
    process_fixed<10>(1);
    process_fixed<100>(1);

    std::cout << "\n=== Method 2: vector with reserve ===\n";
    process_dynamic(10, 1);
    process_dynamic(10000, 1);

    std::cout << "\n=== Method 3: Small buffer optimization ===\n";
    process_sbo(100, 1);     // Uses stack
    process_sbo(1000, 1);    // Falls back to heap

    std::cout << "\n=== Comparison ===\n";
    std::cout << "std::array:     Stack, compile-time size, zero overhead\n";
    std::cout << "vector+reserve: Heap, runtime size, one allocation\n";
    std::cout << "SBO pattern:    Stack when small, heap when large\n";
    std::cout << "alloca/VLA:     Non-standard, no bounds check — AVOID\n";

    return 0;
}

```

---

## Notes

- Stack allocation is ~100x faster than heap — prefer it for small, short-lived objects.
- The compiler can often optimize stack objects better (escape analysis, register allocation).
- `std::array` is the modern C++ replacement for C arrays — same performance, safer interface.
- Use `vector::reserve()` to avoid multiple reallocations when the size is known.
- The small buffer optimization (SBO) pattern is used internally by `std::string`, `std::function`, and others.
