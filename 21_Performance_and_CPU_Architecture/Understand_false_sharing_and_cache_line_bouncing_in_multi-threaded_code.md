# Understand false sharing and cache line bouncing in multi-threaded code

**Category:** Performance & CPU Architecture  
**Item:** #629  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/thread/hardware_destructive_interference_size>  

---

## Topic Overview

False sharing occurs when threads on different cores modify variables that happen to share a cache line (typically 64 bytes). Each write invalidates the cache line on other cores, causing expensive bouncing.

```cpp

Cache line (64 bytes):
+---------------------------+
| counter_A | counter_B     |  <- Thread 0 writes A, Thread 1 writes B
+---------------------------+
     |             |
  Core 0        Core 1
  writes A      writes B
     |             |
  Invalidate!   Invalidate!
     <----------->           <- cache line ping-pongs between cores
                               ~40-100 cycles per ping-pong!

```

| Term | Description |
| --- | --- |
| Cache line | Smallest unit of cache transfer (typically 64 bytes) |
| False sharing | Different variables on same cache line modified by different cores |
| True sharing | Same variable modified by multiple cores |
| `hardware_destructive_interference_size` | C++17: minimum offset to avoid false sharing |
| `hardware_constructive_interference_size` | C++17: max size for true sharing benefit |

---

## Self-Assessment

### Q1: Demonstrate false sharing

```cpp

#include <iostream>
#include <thread>
#include <vector>
#include <chrono>
#include <atomic>

// FALSE SHARING: two counters on the SAME cache line
struct SharedCounters {
    long long counter_a;  // offset 0 (8 bytes)
    long long counter_b;  // offset 8 (same 64-byte cache line!)
};

void increment(long long& counter, int iterations) {
    for (int i = 0; i < iterations; ++i) {
        counter += i; // Each write invalidates the other core's cache line!
    }
}

int main() {
    constexpr int ITERS = 100'000'000;
    SharedCounters shared{};

    auto t0 = std::chrono::high_resolution_clock::now();

    std::thread t1(increment, std::ref(shared.counter_a), ITERS);
    std::thread t2(increment, std::ref(shared.counter_b), ITERS);
    t1.join();
    t2.join();

    auto t1_end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1_end - t0);

    std::cout << "False sharing (adjacent): " << ms.count() << " ms\n";
    std::cout << "counter_a: " << shared.counter_a << '\n';
    std::cout << "counter_b: " << shared.counter_b << '\n';

    // Both counters are on the same 64-byte cache line.
    // Core 0 writes counter_a -> invalidates Core 1's copy
    // Core 1 writes counter_b -> invalidates Core 0's copy
    // Result: ~3-10x slower than without false sharing!
    //
    // Note: even though the threads access DIFFERENT variables,
    // the cache coherence protocol (MESI/MOESI) operates on
    // entire cache lines, not individual bytes.
}

```

### Q2: Fix with padding to separate cache lines

```cpp

#include <iostream>
#include <thread>
#include <chrono>
#include <new>  // hardware_destructive_interference_size

// Option 1: Manual padding
struct PaddedCounters {
    alignas(64) long long counter_a;  // own cache line
    alignas(64) long long counter_b;  // own cache line
};

// Option 2: Using C++17 constant (if supported)
// constexpr size_t CACHE_LINE = std::hardware_destructive_interference_size;
// Many compilers don't define it yet; use 64 directly.
constexpr size_t CACHE_LINE = 64;

struct PaddedCountersV2 {
    alignas(CACHE_LINE) long long counter_a;
    alignas(CACHE_LINE) long long counter_b;
};

// BAD: no padding (false sharing)
struct BadCounters {
    long long counter_a;
    long long counter_b;
};

void increment(long long& counter, int iters) {
    for (int i = 0; i < iters; ++i) counter += i;
}

int main() {
    constexpr int ITERS = 100'000'000;

    auto bench = [&](auto& counters, const char* label) {
        counters.counter_a = 0;
        counters.counter_b = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        std::thread t1(increment, std::ref(counters.counter_a), ITERS);
        std::thread t2(increment, std::ref(counters.counter_b), ITERS);
        t1.join(); t2.join();
        auto t1_end = std::chrono::high_resolution_clock::now();
        auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1_end - t0);
        std::cout << label << ": " << ms.count() << " ms\n";
    };

    BadCounters bad{};
    PaddedCounters padded{};

    bench(bad,    "False sharing (adjacent)");
    bench(padded, "Fixed (padded to 64B)   ");

    std::cout << "\nSizes:\n";
    std::cout << "  BadCounters:    " << sizeof(BadCounters) << " bytes\n";    // 16
    std::cout << "  PaddedCounters: " << sizeof(PaddedCounters) << " bytes\n"; // 128
    // Typical results:
    // False sharing: ~800 ms
    // Fixed:         ~120 ms  (6-7x faster!)
    // Cost: 112 extra bytes per pair. Usually worth it.
}

```

### Q3: Detecting false sharing with `perf c2c`

```cpp

#include <iostream>
#include <thread>
#include <vector>
#include <atomic>

// Program designed to exhibit false sharing for perf c2c detection.
struct FalseSharingDemo {
    std::atomic<long long> counters[8];  // 8 atomics, 64 bytes total
    // All 8 counters fit on ONE cache line!
};

void worker(std::atomic<long long>& counter, int iters) {
    for (int i = 0; i < iters; ++i) {
        counter.fetch_add(1, std::memory_order_relaxed);
    }
}

int main() {
    FalseSharingDemo demo{};
    constexpr int ITERS = 10'000'000;

    std::vector<std::thread> threads;
    for (int i = 0; i < 4; ++i) {
        threads.emplace_back(worker, std::ref(demo.counters[i]), ITERS);
    }
    for (auto& t : threads) t.join();

    for (int i = 0; i < 4; ++i)
        std::cout << "Counter " << i << ": " << demo.counters[i] << '\n';

    // DETECTING with perf c2c:
    //
    // Step 1: Record cache-to-cache transfers
    //   perf c2c record -g ./false_sharing_demo
    //
    // Step 2: Analyze
    //   perf c2c report --stdio
    //
    // Output shows:
    //   =================================================
    //   Shared Data Cache Line Table
    //   =================================================
    //   Total records: 12345
    //   Total HITM:    8901     <-- cache-to-cache transfers (false sharing!)
    //
    //   Cacheline    HITM%   Address    Symbol
    //   0x7f1234560  89.1%   0x...      FalseSharingDemo::counters
    //   ^-- This line shows the hot cache line causing false sharing
    //
    // HITM = Hit In Modified (another core's cache)
    // High HITM% = false sharing!
    //
    // Fix: pad each atomic to its own cache line:
    //   struct alignas(64) PaddedAtomic {
    //       std::atomic<long long> value;
    //   };
}

```

---

## Notes

- Cache line size is 64 bytes on x86/ARM; use `alignas(64)` for padding.
- `std::hardware_destructive_interference_size` (C++17) provides the portable constant, but many compilers don't implement it.
- False sharing is invisible in profilers that only show CPU time — use `perf c2c` specifically.
- Common false sharing sites: thread-local counters in arrays, adjacent atomics, struct members accessed by different threads.
- `perf stat -e L1-dcache-load-misses` shows elevated cache miss rate from false sharing.
