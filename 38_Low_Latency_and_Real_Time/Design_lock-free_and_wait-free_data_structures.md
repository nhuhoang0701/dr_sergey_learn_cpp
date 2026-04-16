# Design Lock-Free and Wait-Free Data Structures

**Category:** Low Latency & Real-Time C++  
**Standard:** C++20 / C++23  
**Reference:** [P0233R0 — Hazard Pointers](https://wg21.link/p0233), [C++ Concurrency in Action, 2nd Ed.](https://www.manning.com/books/c-plus-plus-concurrency-in-action-second-edition)  

---

## Topic Overview

Lock-free data structures guarantee that at least one thread makes progress in any execution, regardless of scheduling. Wait-free structures go further: every thread completes its operation in a bounded number of steps. These properties make them essential in low-latency systems where mutex contention introduces unpredictable tail latency.

The foundation is the **compare-and-swap (CAS)** primitive — `std::atomic::compare_exchange_weak/strong`. A CAS loop speculatively prepares a new state, then atomically publishes it only if no other thread intervened. Weak CAS may fail spuriously but is cheaper on LL/SC architectures (ARM, POWER); strong CAS never fails spuriously and is preferred when the loop body is expensive.

Memory reclamation is the hardest problem. A lock-free pop may read a node that another thread is about to free. Solutions include epoch-based reclamation, hazard pointers (standardized in C++26 as `std::hazard_pointer`), and RCU-style quiescent-state tracking.

| Property | Mutex-Based | Lock-Free | Wait-Free |
| --- | --- | --- | --- |
| Progress guarantee | None (deadlock possible) | System-wide | Per-thread bounded |
| Typical latency | Low contention: fast; high contention: spikes | Consistent, CAS retry overhead | Most consistent |
| Complexity | Low | High | Very high |
| Memory reclamation | Automatic (scope) | Epoch / Hazard pointers | Epoch / Hazard pointers |
| Use case | General purpose | Hot-path queues, stacks | Hard real-time counters |

```cpp

Thread A ──► read top ──► prepare new_node ──► CAS(top, old, new) ──► success
                                                    │ fail
                                                    ▼
                                               retry (reload top)

```

---

## Self-Assessment

### Q1: Implement a lock-free stack with `push` and `try_pop` using CAS, handling the ABA problem with a tagged pointer

```cpp

#include <atomic>
#include <cstdint>
#include <optional>
#include <new>

template <typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        explicit Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    // Tagged pointer: pack a counter into the upper 16 bits
    // (assumes 48-bit virtual addresses on x86-64)
    struct TaggedPtr {
        Node* ptr;
        uint16_t tag;
    };

    static_assert(std::atomic<TaggedPtr>::is_always_lock_free,
                  "TaggedPtr must be lock-free for ABA protection");

    std::atomic<TaggedPtr> head_{TaggedPtr{nullptr, 0}};

public:
    void push(T val) {
        Node* new_node = new Node(std::move(val));
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);
        TaggedPtr new_head;
        do {
            new_node->next = old_head.ptr;
            new_head = TaggedPtr{new_node, static_cast<uint16_t>(old_head.tag + 1)};
        } while (!head_.compare_exchange_weak(old_head, new_head,
                    std::memory_order_release, std::memory_order_relaxed));
    }

    std::optional<T> try_pop() {
        TaggedPtr old_head = head_.load(std::memory_order_acquire);
        TaggedPtr new_head;
        do {
            if (!old_head.ptr) return std::nullopt;
            new_head = TaggedPtr{old_head.ptr->next,
                                 static_cast<uint16_t>(old_head.tag + 1)};
        } while (!head_.compare_exchange_weak(old_head, new_head,
                    std::memory_order_acq_rel, std::memory_order_acquire));
        T val = std::move(old_head.ptr->data);
        delete old_head.ptr;  // simplified; production needs deferred reclamation
        return val;
    }

    ~LockFreeStack() {
        while (try_pop()) {}
    }
};

// Usage
#include <thread>
#include <vector>
#include <cassert>

int main() {
    LockFreeStack<int> stack;
    constexpr int N = 100'000;

    std::thread producer([&] {
        for (int i = 0; i < N; ++i) stack.push(i);
    });
    std::thread consumer([&] {
        int count = 0;
        while (count < N) {
            if (stack.try_pop()) ++count;
        }
    });
    producer.join();
    consumer.join();
}

```

### Q2: Build a wait-free atomic counter with `fetch_add` and a bounded-time `snapshot()` that returns the exact value with no contention spike

```cpp

#include <atomic>
#include <array>
#include <numeric>
#include <thread>
#include <cstdint>
#include <cassert>

// Distributed counter: each thread writes to its own cache line,
// snapshot reads all slots — wait-free for both increment and read.
class WaitFreeCounter {
    static constexpr int kMaxThreads = 64;
    struct alignas(64) Slot {  // one cache line per slot
        std::atomic<int64_t> value{0};
    };
    std::array<Slot, kMaxThreads> slots_{};
    std::atomic<int> next_slot_{0};

    static thread_local int my_slot_;

    int get_slot() {
        if (my_slot_ < 0) {
            my_slot_ = next_slot_.fetch_add(1, std::memory_order_relaxed);
            assert(my_slot_ < kMaxThreads);
        }
        return my_slot_;
    }

public:
    // Wait-free: single atomic fetch_add on thread-local slot
    void increment(int64_t delta = 1) {
        slots_[get_slot()].value.fetch_add(delta, std::memory_order_relaxed);
    }

    // Wait-free: bounded loop (kMaxThreads iterations)
    int64_t snapshot() const {
        int64_t sum = 0;
        int n = next_slot_.load(std::memory_order_acquire);
        for (int i = 0; i < n; ++i) {
            sum += slots_[i].value.load(std::memory_order_relaxed);
        }
        return sum;
    }
};

thread_local int WaitFreeCounter::my_slot_ = -1;

int main() {
    WaitFreeCounter counter;
    constexpr int kThreads = 8;
    constexpr int kOps = 1'000'000;

    std::vector<std::thread> threads;
    for (int t = 0; t < kThreads; ++t) {
        threads.emplace_back([&] {
            for (int i = 0; i < kOps; ++i) counter.increment();
        });
    }
    for (auto& th : threads) th.join();

    assert(counter.snapshot() == int64_t(kThreads) * kOps);
}

```

### Q3: Implement exponential backoff for a CAS retry loop and measure its impact on throughput under contention

```cpp

#include <atomic>
#include <thread>
#include <vector>
#include <chrono>
#include <cstdio>
#include <immintrin.h>  // _mm_pause

struct BackoffPolicy {
    int spin_min = 1;
    int spin_max = 1024;

    void operator()() {
        thread_local int spins = spin_min;
        for (int i = 0; i < spins; ++i) {
            _mm_pause();  // yield execution resources to sibling hyperthread
        }
        spins = std::min(spins * 2, spin_max);  // exponential backoff
    }

    void reset() {
        // reset on successful CAS — called by user
    }
};

std::atomic<int64_t> shared_counter{0};

template <bool UseBackoff>
void increment_loop(int ops) {
    BackoffPolicy backoff;
    for (int i = 0; i < ops; ++i) {
        int64_t old_val = shared_counter.load(std::memory_order_relaxed);
        while (!shared_counter.compare_exchange_weak(old_val, old_val + 1,
                std::memory_order_relaxed)) {
            if constexpr (UseBackoff) backoff();
        }
    }
}

int main() {
    constexpr int kThreads = 8;
    constexpr int kOps = 500'000;

    auto bench = [&](auto fn, const char* label) {
        shared_counter.store(0);
        auto t0 = std::chrono::high_resolution_clock::now();
        std::vector<std::thread> threads;
        for (int i = 0; i < kThreads; ++i)
            threads.emplace_back(fn, kOps);
        for (auto& t : threads) t.join();
        auto t1 = std::chrono::high_resolution_clock::now();
        double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
        std::printf("%-20s %8.2f ms  counter=%lld\n", label, ms,
                    (long long)shared_counter.load());
    };

    bench(increment_loop<false>, "No backoff");
    bench(increment_loop<true>,  "Exp backoff");
}

```

---

## Notes

- **Weak vs strong CAS**: Prefer `compare_exchange_weak` inside loops on ARM/POWER; on x86, both compile identically.
- **ABA problem**: Solved by tagged pointers (DWCAS), hazard pointers, or epoch-based reclamation — never by ignoring it.
- **Wait-free counters** via thread-local slots are the easiest wait-free structure; wait-free queues are research-grade complexity.
- **Memory ordering**: Lock-free structures almost always need at least `acquire`/`release`; `relaxed` is only correct for independent counters.
- **Testing**: Use thread sanitizer (`-fsanitize=thread`) and stress tests with `taskset` to force contention on fewer cores.
- **`std::atomic::wait/notify`** (C++20) can replace spin loops when bounded latency is less critical than power efficiency.
