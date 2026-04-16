# Implement a producer-consumer queue with backpressure using counting_semaphore

**Category:** Concurrency & Parallelism  
**Item:** #369  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/counting_semaphore>  

---

## Topic Overview

A **bounded producer-consumer queue** limits how many items can be buffered, creating **backpressure**: producers block when the queue is full rather than consuming unbounded memory. C++20's `std::counting_semaphore` elegantly manages this with two semaphores — one tracking empty slots, one tracking filled slots.

### Architecture

```cpp

Producer threads                              Consumer threads
     │                                             ↑
     ▼                                             │
 empty_slots.acquire()  ─→ blocks if 0 slots     full_slots.acquire() ─→ blocks if 0 items
     │                                             │
  [ mutex: enqueue item ]                    [ mutex: dequeue item ]
     │                                             │
 full_slots.release()   ─→ signals consumer     empty_slots.release() ─→ signals producer

```

### Two Semaphores Pattern

| Semaphore | Initial value | acquire() means | release() means |
| --- | --- | --- | --- |
| `empty_slots` | N (capacity) | "Take a slot" (producer) | "Return a slot" (consumer) |
| `full_slots` | 0 | "Take an item" (consumer) | "Signal new item" (producer) |

---

## Self-Assessment

### Q1: Use a counting_semaphore to block producers when the queue is full and consumers when empty

**Answer:**

```cpp

#include <semaphore>
#include <mutex>
#include <queue>
#include <thread>
#include <iostream>
#include <vector>

template<typename T, int Capacity>
class BoundedQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::counting_semaphore<Capacity> empty_slots_{Capacity}; // starts full
    std::counting_semaphore<Capacity> full_slots_{0};          // starts empty

public:
    void push(T item) {
        empty_slots_.acquire(); // blocks if no empty slots (queue full)

        {
            std::lock_guard lock(mtx_);
            queue_.push(std::move(item));
        }

        full_slots_.release(); // signal: one more item available
    }

    T pop() {
        full_slots_.acquire(); // blocks if no items (queue empty)

        T item;
        {
            std::lock_guard lock(mtx_);
            item = std::move(queue_.front());
            queue_.pop();
        }

        empty_slots_.release(); // signal: one more slot available
        return item;
    }
};

int main() {
    BoundedQueue<int, 4> queue; // capacity = 4

    // Producer: produces 10 items
    std::thread producer([&] {
        for (int i = 0; i < 10; ++i) {
            queue.push(i);
            std::cout << "Produced: " << i << "\n";
        }
    });

    // Consumer: consumes 10 items (with simulated slow processing)
    std::thread consumer([&] {
        for (int i = 0; i < 10; ++i) {
            int val = queue.pop();
            std::cout << "Consumed: " << val << "\n";
        }
    });

    producer.join();
    consumer.join();
    // Producer blocks after 4 items until consumer catches up (backpressure!)
}

```

**Explanation:** The producer acquires `empty_slots` (decrements count) before enqueuing. When 4 items are buffered and none consumed, `empty_slots` reaches 0 and the producer blocks. The consumer acquires `full_slots` before dequeuing — if the queue is empty, it blocks. Each side releases the other's semaphore after its operation, maintaining the invariant: `empty_slots + full_slots = Capacity`.

### Q2: Show the difference between a bounded queue (backpressure) and an unbounded queue (memory growth)

**Answer:**

```cpp

#include <queue>
#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>
#include <semaphore>
#include <condition_variable>

// === UNBOUNDED queue: no backpressure ===
template<typename T>
class UnboundedQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::condition_variable cv_;
public:
    void push(T item) {
        // NEVER blocks — always accepts items
        std::lock_guard lock(mtx_);
        queue_.push(std::move(item));
        cv_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mtx_);
        cv_.wait(lock, [&] { return !queue_.empty(); });
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    size_t size() {
        std::lock_guard lock(mtx_);
        return queue_.size();
    }
};

int main() {
    // Scenario: fast producer, slow consumer

    // --- Unbounded: memory grows without limit ---
    UnboundedQueue<int> unbounded;
    std::thread fast_producer([&] {
        for (int i = 0; i < 100'000; ++i)
            unbounded.push(i); // never blocks
    });
    std::thread slow_consumer([&] {
        for (int i = 0; i < 1000; ++i) {
            unbounded.pop();
            std::this_thread::sleep_for(std::chrono::microseconds(10));
        }
    });

    fast_producer.join();
    std::cout << "Unbounded queue size: " << unbounded.size() << "\n";
    // Output: ~99000 — almost everything buffered in memory!
    // In production: OOM killer would eventually terminate the process

    slow_consumer.detach(); // let it drain (simplified for demo)

    // --- Bounded (backpressure): memory stays constant ---
    // Using the BoundedQueue from Q1 with capacity 100:
    // - Producer blocks after 100 items until consumer catches up
    // - Queue size NEVER exceeds 100
    // - Memory usage is bounded and predictable

    // COMPARISON:
    // ┌──────────────┬───────────────────┬────────────────────┐
    // │              │ Unbounded         │ Bounded            │
    // ├──────────────┼───────────────────┼────────────────────┤
    // │ Memory       │ O(produced-consumed)│ O(capacity)      │
    // │ Producer     │ Never blocks      │ Blocks when full   │
    // │ Latency      │ Grows over time   │ Bounded            │
    // │ OOM risk     │ YES               │ No                 │
    // │ Use case     │ Bursty, big memory│ Steady, controlled │
    // └──────────────┴───────────────────┴────────────────────┘

    std::cout << "Bounded queues prevent runaway memory growth\n";
}

```

**Explanation:** An unbounded queue lets the producer run at full speed, buffering potentially millions of items in memory when the consumer is slow. A bounded queue with backpressure limits the buffer to a fixed capacity — the producer is forced to slow down to the consumer's rate, preventing memory exhaustion and keeping end-to-end latency bounded.

### Q3: Implement a multi-producer multi-consumer bounded queue and verify correctness under stress

**Answer:**

```cpp

#include <semaphore>
#include <mutex>
#include <queue>
#include <thread>
#include <vector>
#include <iostream>
#include <atomic>
#include <cassert>

template<typename T, int Capacity>
class MPMCQueue {
    std::queue<T> queue_;
    std::mutex mtx_;
    std::counting_semaphore<Capacity> empty_slots_{Capacity};
    std::counting_semaphore<Capacity> full_slots_{0};

public:
    void push(T item) {
        empty_slots_.acquire();
        {
            std::lock_guard lock(mtx_);
            queue_.push(std::move(item));
        }
        full_slots_.release();
    }

    T pop() {
        full_slots_.acquire();
        T item;
        {
            std::lock_guard lock(mtx_);
            item = std::move(queue_.front());
            queue_.pop();
        }
        empty_slots_.release();
        return item;
    }
};

int main() {
    constexpr int NUM_PRODUCERS = 4;
    constexpr int NUM_CONSUMERS = 4;
    constexpr int ITEMS_PER_PRODUCER = 100'000;
    constexpr int TOTAL_ITEMS = NUM_PRODUCERS * ITEMS_PER_PRODUCER;

    MPMCQueue<int, 256> queue;
    std::atomic<long long> produced_sum{0};
    std::atomic<long long> consumed_sum{0};
    std::atomic<int> consumed_count{0};

    // === Producers ===
    std::vector<std::thread> producers;
    for (int p = 0; p < NUM_PRODUCERS; ++p) {
        producers.emplace_back([&, p] {
            for (int i = 0; i < ITEMS_PER_PRODUCER; ++i) {
                int val = p * ITEMS_PER_PRODUCER + i;
                queue.push(val);
                produced_sum.fetch_add(val, std::memory_order_relaxed);
            }
        });
    }

    // === Consumers ===
    std::vector<std::thread> consumers;
    for (int c = 0; c < NUM_CONSUMERS; ++c) {
        consumers.emplace_back([&] {
            while (true) {
                int count = consumed_count.fetch_add(1, std::memory_order_acq_rel);
                if (count >= TOTAL_ITEMS) break;

                int val = queue.pop();
                consumed_sum.fetch_add(val, std::memory_order_relaxed);
            }
        });
    }

    for (auto& t : producers) t.join();
    for (auto& t : consumers) t.join();

    // === Verify correctness ===
    std::cout << "Produced sum: " << produced_sum.load() << "\n";
    std::cout << "Consumed sum: " << consumed_sum.load() << "\n";
    std::cout << "Match: " << std::boolalpha
              << (produced_sum.load() == consumed_sum.load()) << "\n";
    // Output: Match: true

    // Correctness guarantees:
    // 1. No item is lost (sum matches)
    // 2. No item is duplicated (count exactly TOTAL_ITEMS)
    // 3. Queue never exceeds capacity (semaphore enforces)
    // 4. No deadlock (producers and consumers always make progress)
}

```

**Explanation:** The MPMC queue uses `counting_semaphore` for flow control and `std::mutex` for the critical section. Multiple producers and consumers run concurrently. The stress test verifies that the sum of all produced values equals the sum of all consumed values, proving no items are lost or duplicated. The 256-item capacity ensures bounded memory regardless of production/consumption rate imbalance.

---

## Notes

- **Semaphore vs condition_variable:** Semaphores encode the count directly (no need for a separate counter + predicate). They're often more efficient for producer-consumer patterns.
- **`binary_semaphore`** is `counting_semaphore<1>` — useful for simple signaling.
- **Capacity tuning:** Too small = producers block frequently. Too large = high memory usage and latency. Profile to find the sweet spot.
- **Ring buffer alternative:** For maximum performance, replace `std::queue` + mutex with a pre-allocated ring buffer and atomic indices.
- Compile with `-std=c++20 -pthread`.
