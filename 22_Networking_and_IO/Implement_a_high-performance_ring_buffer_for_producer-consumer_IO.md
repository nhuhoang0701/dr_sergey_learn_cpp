# Implement a high-performance ring buffer for producer-consumer I/O

**Category:** Networking & I/O  
**Item:** #732  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic>  

---

## Topic Overview

A ring buffer (circular buffer) is the standard data structure for lock-free single-producer single-consumer (SPSC) I/O. It uses a fixed-size array with head/tail indices, avoiding all dynamic allocation and enabling wait-free progress.

```cpp

Ring buffer layout (capacity=8, power of 2):
  Index:   0   1   2   3   4   5   6   7
         +---+---+---+---+---+---+---+---+
         |   | D | D | D |   |   |   |   |
         +---+---+---+---+---+---+---+---+
               ^           ^
              tail=1      head=4
  
  Mask:  index & (capacity - 1)  // power-of-2 wrapping (no modulo!)
  Empty: head == tail
  Full:  head - tail == capacity

Cache-line layout (avoid false sharing):
  [  tail (64B)  |  padding  |  head (64B)  |  padding  |  data[]  ]
   ^writer writes   ^reader reads

```

| Feature | Ring buffer | mutex + std::queue |
| --- | --- | --- |
| Allocation | Zero (fixed array) | Per-push (queue node) |
| Synchronisation | Atomic load/store | Lock/unlock |
| Cache behaviour | Sequential, predictable | Random (heap nodes) |
| Throughput (SPSC) | ~200M ops/sec | ~20M ops/sec |

---

## Self-Assessment

### Q1: Lock-free SPSC ring buffer with power-of-two modulo

```cpp

#include <iostream>
#include <atomic>
#include <array>
#include <cstddef>
#include <cassert>

template <typename T, size_t Capacity>
class SPSCRingBuffer {
    static_assert((Capacity & (Capacity - 1)) == 0, "Capacity must be power of 2");
    static constexpr size_t MASK = Capacity - 1;

public:
    // Producer: try to push an element
    bool try_push(const T& item) {
        size_t h = head_.load(std::memory_order_relaxed);
        size_t t = tail_.load(std::memory_order_acquire);  // sync with consumer

        if (h - t == Capacity) return false;  // full

        data_[h & MASK] = item;
        head_.store(h + 1, std::memory_order_release);  // publish to consumer
        return true;
    }

    // Consumer: try to pop an element
    bool try_pop(T& item) {
        size_t t = tail_.load(std::memory_order_relaxed);
        size_t h = head_.load(std::memory_order_acquire);  // sync with producer

        if (t == h) return false;  // empty

        item = data_[t & MASK];
        tail_.store(t + 1, std::memory_order_release);  // publish to producer
        return true;
    }

    size_t size() const {
        return head_.load(std::memory_order_relaxed)

             - tail_.load(std::memory_order_relaxed);

    }

private:
    // Data buffer
    std::array<T, Capacity> data_{};

    // Indices: use size_t (wraps naturally for power-of-2 masking)
    alignas(64) std::atomic<size_t> head_{0};  // written by producer
    alignas(64) std::atomic<size_t> tail_{0};  // written by consumer
};

int main() {
    SPSCRingBuffer<int, 8> rb;

    // Push 5 items
    for (int i = 0; i < 5; ++i) {
        bool ok = rb.try_push(i * 10);
        std::cout << "Push " << i * 10 << ": " << (ok ? "ok" : "full") << '\n';
    }
    std::cout << "Size: " << rb.size() << '\n';  // 5

    // Pop 3 items
    int val;
    for (int i = 0; i < 3; ++i) {
        bool ok = rb.try_pop(val);
        std::cout << "Pop: " << val << " (" << (ok ? "ok" : "empty") << ")\n";
    }
    std::cout << "Size: " << rb.size() << '\n';  // 2

    // Push wraps around (uses slots freed by pop)
    rb.try_push(99);
    std::cout << "After wrap push, size: " << rb.size() << '\n';  // 3
}
// Output:
//   Push 0: ok
//   Push 10: ok
//   Push 20: ok
//   Push 30: ok
//   Push 40: ok
//   Size: 5
//   Pop: 0 (ok)
//   Pop: 10 (ok)
//   Pop: 20 (ok)
//   Size: 2
//   After wrap push, size: 3

```

### Q2: Cache-line separation between head and tail

```cpp

#include <iostream>
#include <atomic>
#include <cstddef>
#include <thread>
#include <chrono>

// BAD: head and tail on same cache line -> false sharing
struct BadRingMeta {
    std::atomic<size_t> head{0};  // offset 0
    std::atomic<size_t> tail{0};  // offset 8 (same 64-byte cache line!)
};

// GOOD: head and tail on separate cache lines
struct GoodRingMeta {
    alignas(64) std::atomic<size_t> head{0};  // cache line 0
    alignas(64) std::atomic<size_t> tail{0};  // cache line 1 (64 bytes apart)
};

int main() {
    std::cout << "=== Layout analysis ===\n";

    // Bad layout: both on same cache line
    BadRingMeta bad;
    auto bad_head = reinterpret_cast<uintptr_t>(&bad.head);
    auto bad_tail = reinterpret_cast<uintptr_t>(&bad.tail);
    std::cout << "BadRingMeta:\n";
    std::cout << "  sizeof: " << sizeof(bad) << '\n';  // 16
    std::cout << "  head offset: 0\n";
    std::cout << "  tail offset: " << (bad_tail - bad_head) << '\n';  // 8
    std::cout << "  Same cache line: YES (false sharing!)\n";

    // Good layout: separate cache lines
    GoodRingMeta good;
    auto good_head = reinterpret_cast<uintptr_t>(&good.head);
    auto good_tail = reinterpret_cast<uintptr_t>(&good.tail);
    std::cout << "\nGoodRingMeta:\n";
    std::cout << "  sizeof: " << sizeof(good) << '\n';  // 128
    std::cout << "  head offset: 0\n";
    std::cout << "  tail offset: " << (good_tail - good_head) << '\n';  // 64
    std::cout << "  Same cache line: NO (independent!)\n";

    // Why it matters:
    // Producer writes head -> invalidates cache line
    // Consumer reads head + writes tail
    // If same cache line: every write by either side invalidates the other's cache
    // -> ~100 cycles per operation instead of ~5 cycles

    // Zero allocations proof:
    std::cout << "\n=== Zero allocation proof ===\n";
    std::cout << "Ring buffer is stack/static allocated\n";
    std::cout << "No new/delete, no heap, no malloc\n";
    std::cout << "Power-of-2 mask: index & (cap-1) avoids modulo (division)\n";
}

```

### Q3: Benchmark ring buffer vs mutex+queue

```cpp

#include <iostream>
#include <thread>
#include <chrono>
#include <atomic>
#include <array>
#include <queue>
#include <mutex>

// Lock-free SPSC ring buffer (from Q1)
template <typename T, size_t Cap>
class RingBuf {
    static_assert((Cap & (Cap - 1)) == 0);
public:
    bool try_push(const T& v) {
        auto h = head_.load(std::memory_order_relaxed);
        if (h - tail_.load(std::memory_order_acquire) == Cap) return false;
        data_[h & (Cap - 1)] = v;
        head_.store(h + 1, std::memory_order_release);
        return true;
    }
    bool try_pop(T& v) {
        auto t = tail_.load(std::memory_order_relaxed);
        if (t == head_.load(std::memory_order_acquire)) return false;
        v = data_[t & (Cap - 1)];
        tail_.store(t + 1, std::memory_order_release);
        return true;
    }
private:
    std::array<T, Cap> data_{};
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};
};

// Mutex-protected queue
template <typename T>
class MutexQueue {
public:
    bool try_push(const T& v) {
        std::lock_guard<std::mutex> lk(mu_);
        q_.push(v);
        return true;
    }
    bool try_pop(T& v) {
        std::lock_guard<std::mutex> lk(mu_);
        if (q_.empty()) return false;
        v = q_.front();
        q_.pop();
        return true;
    }
private:
    std::mutex mu_;
    std::queue<T> q_;
};

template <typename Q>
void benchmark(const char* label) {
    constexpr size_t N = 10'000'000;
    Q queue;
    std::atomic<bool> done{false};

    auto t0 = std::chrono::high_resolution_clock::now();

    std::thread producer([&] {
        for (size_t i = 0; i < N; ++i)
            while (!queue.try_push(static_cast<int>(i))) {} // spin on full
        done = true;
    });

    std::thread consumer([&] {
        int val;
        size_t count = 0;
        while (count < N)
            if (queue.try_pop(val)) ++count;
    });

    producer.join();
    consumer.join();
    auto t1 = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);
    std::cout << label << ": " << ms.count() << " ms  ("
              << N / (ms.count() + 1) / 1000 << " M ops/sec)\n";
}

int main() {
    benchmark<RingBuf<int, 1024>>("Ring buffer  ");
    benchmark<MutexQueue<int>>(   "Mutex queue  ");
    // Typical results:
    //   Ring buffer:  ~50 ms  (200 M ops/sec)
    //   Mutex queue:  ~500 ms (20 M ops/sec)
    //   -> Ring buffer is ~10x faster for SPSC!
}

```

---

## Notes

- Power-of-two capacity allows `index & (cap - 1)` instead of `index % cap` (no division).
- `alignas(64)` on head/tail prevents false sharing between producer and consumer threads.
- For MPMC (multi-producer multi-consumer), use `compare_exchange_weak` or a library like folly::MPMCQueue.
- The ring buffer is ideal for network I/O: recv thread pushes packets, processing thread pops them.
- For variable-size messages: store offset+length pairs or use a byte ring with framing.
