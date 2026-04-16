# Avoid false sharing with cache-line padding in concurrent data structures

**Category:** Concurrency & Parallelism  
**Item:** #187  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size>  

---

## Topic Overview

**False sharing** occurs when two threads access different variables that happen to reside on the same CPU cache line. Even though there's no logical data race, the hardware cache coherency protocol (e.g., MESI) forces expensive cache invalidations between cores, destroying performance.

### How False Sharing Works

```cpp

Cache Line (64 bytes typically):
┌────────────────────────────────────────────────────────────┐
│  counter_a (4 bytes) │  counter_b (4 bytes) │  padding... │
└────────────────────────────────────────────────────────────┘
        ↑ Thread 1 writes          ↑ Thread 2 writes

When Thread 1 writes counter_a:
  → Core 2's cache line is INVALIDATED
  → Core 2 must re-fetch the ENTIRE line from L3/memory
  → Even though counter_b hasn't changed!

```

### The Fix: Pad Each Variable to Its Own Cache Line

```cpp

┌────────────────────────────────────────────────────────────┐
│  counter_a (4 bytes)  │  padding (60 bytes)               │
└────────────────────────────────────────────────────────────┘
┌────────────────────────────────────────────────────────────┐
│  counter_b (4 bytes)  │  padding (60 bytes)               │
└────────────────────────────────────────────────────────────┘
  ↑ Thread 1's cache line    ↑ Thread 2's cache line  — independent!

```

### C++17 Constants

| Constant | Meaning | Typical value |
| --- | --- | --- |
| `hardware_destructive_interference_size` | Min offset to AVOID false sharing | 64 |
| `hardware_constructive_interference_size` | Max size to PROMOTE true sharing | 64 |

---

## Self-Assessment

### Q1: Show a benchmark where two threads update adjacent atomic<int>s and suffer false sharing

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>

struct NoPadding {
    std::atomic<int> a{0};  // Adjacent — likely same cache line!
    std::atomic<int> b{0};
};

constexpr int ITERATIONS = 100'000'000;

void bench(const char* label, auto& data) {
    data.a = 0;
    data.b = 0;

    auto start = std::chrono::steady_clock::now();

    std::thread t1([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.a.fetch_add(1, std::memory_order_relaxed);
    });
    std::thread t2([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.b.fetch_add(1, std::memory_order_relaxed);
    });

    t1.join();
    t2.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << label << ": " << ms << " ms\n";
}

int main() {
    NoPadding np;
    bench("False sharing (adjacent)", np);
    // Typical output: ~800-1200 ms on modern x86
    // Cache line bouncing between cores causes massive slowdown

    // Compare: single-threaded baseline
    auto start = std::chrono::steady_clock::now();
    std::atomic<int> single{0};
    for (int i = 0; i < ITERATIONS * 2; ++i)
        single.fetch_add(1, std::memory_order_relaxed);
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();
    std::cout << "Single thread (2x iters): " << ms << " ms\n";
    // Often FASTER than the false-sharing version despite doing same total work!
}

```

**Explanation:** Even though `a` and `b` are independent atomics, they're adjacent in memory (likely same cache line). When Thread 1 writes `a`, it invalidates Thread 2's cache line containing `b`, forcing a re-fetch. This "ping-pong" between cores can make concurrent code slower than single-threaded.

### Q2: Fix it by aligning each counter to alignas(std::hardware_destructive_interference_size)

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <chrono>
#include <iostream>
#include <new> // for hardware_destructive_interference_size

struct Padded {
    alignas(std::hardware_destructive_interference_size)
    std::atomic<int> a{0};

    alignas(std::hardware_destructive_interference_size)
    std::atomic<int> b{0};
};

// Fallback for compilers that don't define it:
// #ifndef __cpp_lib_hardware_interference_size
// constexpr std::size_t hardware_destructive_interference_size = 64;
// #endif

constexpr int ITERATIONS = 100'000'000;

int main() {
    // Verify padding:
    std::cout << "sizeof(NoPadding struct): " << 2 * sizeof(std::atomic<int>) << " bytes\n";
    std::cout << "sizeof(Padded):           " << sizeof(Padded) << " bytes\n";
    std::cout << "Cache line size:          "
              << std::hardware_destructive_interference_size << " bytes\n";
    // Typical output:
    // sizeof(NoPadding struct): 8 bytes
    // sizeof(Padded):           128 bytes   (each on its own cache line)
    // Cache line size:          64 bytes

    Padded data;

    auto start = std::chrono::steady_clock::now();

    std::thread t1([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.a.fetch_add(1, std::memory_order_relaxed);
    });
    std::thread t2([&] {
        for (int i = 0; i < ITERATIONS; ++i)
            data.b.fetch_add(1, std::memory_order_relaxed);
    });

    t1.join();
    t2.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << "Padded (no false sharing): " << ms << " ms\n";
    // Typical: ~300-400 ms — 2-4x faster than the false-sharing version!

    // The improvement comes from each core's cache being independent:
    //   Before: Core 1 writes a → invalidates Core 2's line → Core 2 refetch
    //   After:  Core 1 writes a → only its own cache line → Core 2 unaffected
}

```

**Explanation:** `alignas(std::hardware_destructive_interference_size)` forces each counter to start at a cache-line boundary, guaranteed to be on separate cache lines. This eliminates the false sharing ping-pong. The cost is extra memory (128 bytes instead of 8), which is a worthwhile trade-off for frequently-written concurrent data.

### Q3: Explain std::hardware_constructive_interference_size and when to use it instead

**Answer:**

```cpp

#include <new> // hardware_constructive_interference_size
#include <iostream>
#include <atomic>

// hardware_DESTRUCTIVE_interference_size:
//   "Keep these APART — they're accessed by DIFFERENT threads"
//   Use: padding between thread-private counters
//
// hardware_CONSTRUCTIVE_interference_size:
//   "Keep these TOGETHER — they're accessed by the SAME thread"
//   Use: ensure data accessed together fits in ONE cache line

// Example: A node that one thread accesses frequently
struct Node {
    int key;
    int value;
    Node* next;
    // Total ~20 bytes on 64-bit — fits in one cache line, good for cache hits
};

// Use constructive size to verify data fits one cache line:
static_assert(sizeof(Node) <= std::hardware_constructive_interference_size,
              "Node should fit in a single cache line for optimal access");

// Example: Producer token and its counter should be on the SAME line
struct alignas(std::hardware_constructive_interference_size) ProducerState {
    std::atomic<int> sequence{0}; // These two are always accessed together
    int cached_value{0};           // by the same producer thread
    // Both fit in one cache line → single cache fetch = fast
};

// But two ProducerStates for DIFFERENT threads need DESTRUCTIVE padding:
struct alignas(std::hardware_destructive_interference_size) PaddedProducer {
    ProducerState state;
};

int main() {
    std::cout << "Destructive interference size: "
              << std::hardware_destructive_interference_size << "\n";
    std::cout << "Constructive interference size: "
              << std::hardware_constructive_interference_size << "\n";
    // Both are typically 64 on x86 (same cache line size)

    // Summary:
    // DESTRUCTIVE: alignas to SEPARATE  → avoid false sharing between threads
    // CONSTRUCTIVE: pack together within → promote true sharing within a thread

    // Real-world example: thread pool with per-thread work queues
    // - Queue header (head + tail pointers) → constructive (same thread reads both)
    // - Different threads' queues → destructive (pad between them)
}

```

**Explanation:**

- **`hardware_destructive_interference_size`** = minimum spacing to place data on **separate** cache lines, preventing false sharing between threads.
- **`hardware_constructive_interference_size`** = maximum size of data that should be on the **same** cache line, promoting locality for single-thread access.

On most architectures both equal 64 bytes (the typical L1 cache line size), but they represent opposite design intents.

---

## Notes

- **Portability:** Not all compilers define these constants (GCC/Clang require `-std=c++17`; some older versions may not support them). Use `#ifdef __cpp_lib_hardware_interference_size` and fall back to 64.
- **Memory trade-off:** Padding increases struct size (e.g., 8 bytes → 128 bytes). Only pad data that is frequently written by different threads.
- **`perf c2c`:** Linux tool to detect false sharing at runtime.
- **Read-only data doesn't cause false sharing:** If all threads only read, cache lines are shared without invalidation (MESI Shared state).
- Compile with `-std=c++17 -O2 -pthread`.

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
