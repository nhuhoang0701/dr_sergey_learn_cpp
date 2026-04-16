# Understand std::hardware_destructive_interference_size and alignment for concurrent objects

**Category:** Concurrency & Parallelism  
**Item:** #245  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size>  

---

## Topic Overview

C++17 provides two constants in `<new>` that expose cache-line information to portable C++ code:

| Constant | Meaning | Typical Value |
| --- | --- | --- |
| `hardware_destructive_interference_size` | Minimum offset to **avoid** false sharing | 64 bytes (x86), 128 bytes (Apple M1) |
| `hardware_constructive_interference_size` | Maximum size to **promote** true sharing | 64 bytes (x86), 128 bytes (Apple M1) |

### False Sharing Visual

```cpp

WITHOUT alignment (false sharing):
┌─────────────────────────────────────────────────────────────────┐
│ Cache Line (64 bytes)                                           │
│ ┌──────────┐┌──────────┐                                        │
│ │ counter_a ││ counter_b │  ← both on SAME cache line            │
│ └──────────┘└──────────┘                                        │
└─────────────────────────────────────────────────────────────────┘
Thread 1 writes counter_a → invalidates cache line for Thread 2
Thread 2 writes counter_b → invalidates cache line for Thread 1
→ Constant ping-pong, 10-50x slower!

WITH alignment (no false sharing):
┌────────────────────────────────┐ ┌────────────────────────────────┐
│ Cache Line 1 (64 bytes)        │ │ Cache Line 2 (64 bytes)        │
│ ┌──────────┐                   │ │ ┌──────────┐                   │
│ │ counter_a │  (padded)        │ │ │ counter_b │  (padded)        │
│ └──────────┘                   │ │ └──────────┘                   │
└────────────────────────────────┘ └────────────────────────────────┘
Each counter on its OWN cache line → no invalidation traffic

```

---

## Self-Assessment

### Q1: Use alignas(std::hardware_destructive_interference_size) to isolate two atomic counters to different cache lines

**Answer:**

```cpp

#include <atomic>
#include <new>      // hardware_destructive_interference_size
#include <iostream>
#include <thread>
#include <vector>
#include <chrono>

// WITHOUT alignment: both counters likely on the same cache line
struct PackedCounters {
    std::atomic<long long> a{0};
    std::atomic<long long> b{0}; // only 8 bytes after 'a' — same cache line
};

// WITH alignment: each counter on its OWN cache line
struct AlignedCounters {
    alignas(std::hardware_destructive_interference_size)
        std::atomic<long long> a{0};

    alignas(std::hardware_destructive_interference_size)
        std::atomic<long long> b{0};
};

int main() {
    std::cout << "destructive_interference_size = "
              << std::hardware_destructive_interference_size << " bytes\n";
    std::cout << "sizeof(PackedCounters)  = " << sizeof(PackedCounters) << "\n";
    std::cout << "sizeof(AlignedCounters) = " << sizeof(AlignedCounters) << "\n";
    // Output:
    // destructive_interference_size = 64 bytes
    // sizeof(PackedCounters)  = 16    (two 8-byte atomics, adjacent)
    // sizeof(AlignedCounters) = 128   (each padded to 64 bytes)

    // Verify offsets
    AlignedCounters ac;
    auto offset = reinterpret_cast<char*>(&ac.b) -
                  reinterpret_cast<char*>(&ac.a);
    std::cout << "Offset between a and b: " << offset << " bytes\n";
    // Output: Offset between a and b: 64 bytes (or 128 on Apple M1)

    // Quick benchmark to demonstrate the effect
    constexpr int ITERS = 50'000'000;

    auto bench = [](auto& counters, const char* name) {
        counters.a.store(0);
        counters.b.store(0);
        auto start = std::chrono::steady_clock::now();

        std::thread t1([&] {
            for (int i = 0; i < ITERS; ++i)
                counters.a.fetch_add(1, std::memory_order_relaxed);
        });
        std::thread t2([&] {
            for (int i = 0; i < ITERS; ++i)
                counters.b.fetch_add(1, std::memory_order_relaxed);
        });
        t1.join();
        t2.join();

        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << name << ": " << ms << " ms\n";
    };

    PackedCounters packed;
    AlignedCounters aligned;
    bench(packed, "Packed (false sharing)  ");
    bench(aligned, "Aligned (no sharing)   ");
    // Typical output:
    // Packed (false sharing)  : 350 ms
    // Aligned (no sharing)   : 80 ms  (~4x faster)
}

```

**Explanation:** `alignas(hardware_destructive_interference_size)` forces each counter to start at a cache-line boundary and occupy at least one full cache line. This prevents the CPU cache coherence protocol from bouncing the line between cores when different threads write to different counters.

### Q2: Show the false sharing performance degradation before and after alignment

**Answer:**

```cpp

#include <atomic>
#include <new>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>
#include <iomanip>

// Four threads, four counters — worst case for false sharing
struct FalseSharing {
    std::atomic<long long> c[4]; // 4 × 8 = 32 bytes, all in ONE cache line
};

struct NoFalseSharing {
    struct alignas(std::hardware_destructive_interference_size) PaddedCounter {
        std::atomic<long long> value{0};
    };
    PaddedCounter c[4]; // 4 × 64 = 256 bytes, each in its OWN cache line
};

template<typename Counters>
long long benchmark(const char* label) {
    Counters counters{};
    constexpr int ITERS = 20'000'000;

    auto start = std::chrono::steady_clock::now();

    std::vector<std::thread> threads;
    for (int t = 0; t < 4; ++t) {
        threads.emplace_back([&, t] {
            auto& counter = [&]() -> std::atomic<long long>& {
                if constexpr (std::is_same_v<Counters, FalseSharing>)
                    return counters.c[t];
                else
                    return counters.c[t].value;
            }();
            for (int i = 0; i < ITERS; ++i)
                counter.fetch_add(1, std::memory_order_relaxed);
        });
    }
    for (auto& t : threads) t.join();

    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    std::cout << std::setw(30) << label << ": " << ms << " ms"
              << "  (sizeof = " << sizeof(Counters) << ")\n";
    return ms;
}

int main() {
    auto slow = benchmark<FalseSharing>("False sharing (packed)");
    auto fast = benchmark<NoFalseSharing>("No false sharing (aligned)");

    std::cout << "Speedup: " << std::fixed << std::setprecision(1)
              << (double)slow / fast << "x\n";

    // Typical output (4 cores):
    //     False sharing (packed): 800 ms  (sizeof = 32)
    //  No false sharing (aligned): 120 ms  (sizeof = 256)
    // Speedup: 6.7x
    //
    // The 6-7x speedup comes from eliminating cache-line bouncing.
    // Each core's L1 cache can keep its counter without invalidation.
}

```

**Explanation:** With 4 threads writing to adjacent 8-byte atomics, all four fit in one 64-byte cache line. Every write by any thread forces all other cores to invalidate and re-fetch the line. With `alignas` padding, each counter gets its own cache line — cores never interfere with each other.

### Q3: Explain that the value is implementation-defined and must not be assumed to be 64

**Answer:**

```cpp

The value of hardware_destructive_interference_size is:

- Implementation-defined (set by the compiler, not the hardware)
- A compile-time constant (constexpr)
- NOT necessarily 64

Known values across platforms:
┌────────────────────┬───────────────────────┬────────┐
│ Platform           │ destructive_interf.   │ constr.│
├────────────────────┼───────────────────────┼────────┤
│ x86_64 (Intel/AMD) │ 64                    │ 64     │
│ Apple M1/M2 (ARM)  │ 128                   │ 128    │
│ AWS Graviton (ARM) │ 64                    │ 64     │
│ POWER9             │ 128                   │ 128    │
│ RISC-V             │ typically 64          │ 64     │
└────────────────────┴───────────────────────┴────────┘

```

**Why it matters:**

```cpp

#include <new>
#include <iostream>

// BAD: hardcoded assumption
struct Bad {
    alignas(64) std::atomic<int> a; // wrong on Apple M1 (128-byte lines)
};

// GOOD: portable
struct Good {
    alignas(std::hardware_destructive_interference_size)
        std::atomic<int> a; // correct on ALL platforms
};

// COMPILER ISSUES:
// - GCC: available since GCC 12 (earlier versions may not define it)
// - Clang: may warn about ABI instability
// - MSVC: available since VS 2019
//
// Portable fallback:
#ifdef __cpp_lib_hardware_interference_size
    constexpr std::size_t CACHE_LINE = std::hardware_destructive_interference_size;
#else
    constexpr std::size_t CACHE_LINE = 64; // safe fallback for x86
#endif

int main() {
    std::cout << "hardware_destructive_interference_size = "
              << std::hardware_destructive_interference_size << "\n";
    std::cout << "hardware_constructive_interference_size = "
              << std::hardware_constructive_interference_size << "\n";

    // These are COMPILE-TIME constants — the compiler bakes them in.
    // If you cross-compile for ARM on x86, the VALUE MUST MATCH THE TARGET.
    // The compiler picks the value for the target architecture.

    static_assert(std::hardware_destructive_interference_size >= alignof(std::max_align_t),
        "Cache line must be at least max alignment");

    // Note: constructive_interference_size tells you the maximum size
    // of objects that you WANT on the SAME cache line (for true sharing).
    // Example: a mutex and the data it protects should fit within this size.
}

```

**Key takeaway:** Always use `std::hardware_destructive_interference_size` instead of hardcoding 64. The value varies across architectures, and the constant ensures portability. When the constant isn't available, use a `#ifdef` fallback.

---

## Notes

- **Destructive interference** (avoid sharing): pad objects accessed by *different* threads to separate cache lines.
- **Constructive interference** (promote sharing): keep objects accessed by the *same* thread together within one cache line.
- **Clang warning:** Clang may emit `-Winterference-size` because the value affects ABI and could differ between translation units compiled with different flags.
- **Apple M1/M2:** 128-byte cache lines mean naive `alignas(64)` is insufficient — you'll still get false sharing.
- **Memory cost:** Padding increases structure size (e.g., 16 bytes → 128 bytes). Only pad objects that are *actually* accessed by different threads concurrently.
- Compile with `-std=c++17 -O2 -pthread`.
