# Test concurrent code with ThreadSanitizer and stress testing

**Category:** Testing in Practice

---

## Topic Overview

Concurrency bugs (data races, deadlocks, atomicity violations) are among the hardest to detect because they're **non-deterministic** — they may not reproduce in normal testing. Two complementary strategies: **ThreadSanitizer (TSan)** for deterministic race detection, and **stress testing** for exercising timing-dependent paths.

### Detection Strategies

| Strategy | What It Finds | Deterministic? | Overhead |
| --- | --- | --- | --- |
| **TSan** | Data races, lock ordering | Yes (instruments all accesses) | 5-15x slower |
| **Stress testing** | Timing-dependent bugs, starvation | No (probabilistic) | Varies |
| **Helgrind** (Valgrind) | Races, lock misuse | Yes | 20-100x slower |
| **Code review** | Design-level issues | N/A | Human time |
| **Model checking** | Exhaustive state exploration | Yes | Exponential |

---

## Self-Assessment

### Q1: Write tests that TSan can detect races in, and show how to fix them

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <thread>
#include <mutex>
#include <atomic>
#include <vector>
#include <numeric>

// === BUG: Data race on shared counter ===
class BuggyCounter {
public:
    void increment() { count_++; }  // RACE: no synchronization
    int get() const { return count_; }
private:
    int count_ = 0;  // Shared, mutable, not synchronized
};

TEST(DataRaceTest, BuggyCounterHasRace) {
    BuggyCounter counter;
    constexpr int THREADS = 4;
    constexpr int INCREMENTS = 10000;

    std::vector<std::thread> threads;
    for (int i = 0; i < THREADS; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < INCREMENTS; ++j)
                counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    // Without TSan, this may "pass" with wrong count.
    // With TSan: ERROR: data race detected!
    // EXPECT_EQ(counter.get(), THREADS * INCREMENTS);  // Likely fails
}

// === FIX 1: Mutex-protected counter ===
class MutexCounter {
public:
    void increment() {
        std::lock_guard lock(mtx_);
        count_++;
    }
    int get() const {
        std::lock_guard lock(mtx_);
        return count_;
    }
private:
    mutable std::mutex mtx_;
    int count_ = 0;
};

// === FIX 2: Atomic counter (better for simple ops) ===
class AtomicCounter {
public:
    void increment() { count_.fetch_add(1, std::memory_order_relaxed); }
    int get() const { return count_.load(std::memory_order_relaxed); }
private:
    std::atomic<int> count_{0};
};

TEST(DataRaceTest, AtomicCounterIsCorrect) {
    AtomicCounter counter;
    constexpr int THREADS = 4;
    constexpr int INCREMENTS = 10000;

    std::vector<std::thread> threads;
    for (int i = 0; i < THREADS; ++i) {
        threads.emplace_back([&] {
            for (int j = 0; j < INCREMENTS; ++j)
                counter.increment();
        });
    }
    for (auto& t : threads) t.join();

    EXPECT_EQ(counter.get(), THREADS * INCREMENTS);  // Always correct
}

```

### Q2: Design stress tests for concurrent data structures

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <thread>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <atomic>
#include <chrono>
#include <set>

// Thread-safe queue under test
template<typename T>
class ConcurrentQueue {
public:
    void push(T value) {
        {
            std::lock_guard lock(mtx_);
            queue_.push(std::move(value));
        }
        cv_.notify_one();
    }

    std::optional<T> try_pop() {
        std::lock_guard lock(mtx_);
        if (queue_.empty()) return std::nullopt;
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    T wait_and_pop() {
        std::unique_lock lock(mtx_);
        cv_.wait(lock, [this] { return !queue_.empty(); });
        T value = std::move(queue_.front());
        queue_.pop();
        return value;
    }

    size_t size() const {
        std::lock_guard lock(mtx_);
        return queue_.size();
    }

private:
    mutable std::mutex mtx_;
    std::condition_variable cv_;
    std::queue<T> queue_;
};

// === Stress test: no lost items ===
TEST(ConcurrentQueueStress, NoLostItems) {
    ConcurrentQueue<int> queue;
    constexpr int PRODUCERS = 4;
    constexpr int ITEMS_PER_PRODUCER = 10000;
    constexpr int TOTAL = PRODUCERS * ITEMS_PER_PRODUCER;

    // Producers push unique values
    std::vector<std::thread> producers;
    for (int p = 0; p < PRODUCERS; ++p) {
        producers.emplace_back([&, p] {
            for (int i = 0; i < ITEMS_PER_PRODUCER; ++i)
                queue.push(p * ITEMS_PER_PRODUCER + i);
        });
    }

    // Consumers collect values
    std::atomic<int> consumed{0};
    std::mutex result_mtx;
    std::set<int> results;

    constexpr int CONSUMERS = 4;
    std::vector<std::thread> consumers;
    for (int c = 0; c < CONSUMERS; ++c) {
        consumers.emplace_back([&] {
            while (consumed.load() < TOTAL) {
                auto val = queue.try_pop();
                if (val) {
                    std::lock_guard lock(result_mtx);
                    results.insert(*val);
                    consumed.fetch_add(1);
                }
                // No sleep — tight loop stresses scheduling
            }
        });
    }

    for (auto& t : producers) t.join();
    for (auto& t : consumers) t.join();

    // Verify: every item consumed exactly once
    EXPECT_EQ(results.size(), TOTAL);
}

// === Stress test: producer-consumer with wait ===
TEST(ConcurrentQueueStress, WaitAndPopUnderContention) {
    ConcurrentQueue<int> queue;
    std::atomic<bool> done{false};
    std::vector<int> received;
    std::mutex recv_mtx;

    // Consumer starts BEFORE producer (must not deadlock)
    auto consumer = std::thread([&] {
        while (!done.load() || queue.size() > 0) {
            auto val = queue.try_pop();
            if (val) {
                std::lock_guard lock(recv_mtx);
                received.push_back(*val);
            }
        }
    });

    // Producer sends items with slight delays to vary timing
    for (int i = 0; i < 1000; ++i) {
        queue.push(i);
        if (i % 100 == 0)
            std::this_thread::yield();  // Vary timing
    }
    done.store(true);

    consumer.join();
    EXPECT_EQ(received.size(), 1000);
}

```

### Q3: Show TSan integration patterns and suppression of known issues

**Answer:**

```cpp

// === Annotate intentionally racy code for TSan ===
// Sometimes code has benign races (e.g., performance counters).
// Use TSan annotations to suppress false positives.

#if defined(__SANITIZE_THREAD__) || defined(__has_feature)
#if __has_feature(thread_sanitizer)
#define TSAN_ENABLED 1
#endif
#endif

#ifdef TSAN_ENABLED
#include <sanitizer/tsan_interface.h>
#define TSAN_ANNOTATE_BENIGN_RACE(addr, desc) \
    AnnotateBenignRace(__FILE__, __LINE__, addr, desc)
#else
#define TSAN_ANNOTATE_BENIGN_RACE(addr, desc)
#endif

class PerformanceCounters {
public:
    void record_request() {
        // Benign race: counter is approximate, never used for correctness
        TSAN_ANNOTATE_BENIGN_RACE(&request_count_, "Approximate counter");
        ++request_count_;
    }

    int64_t request_count() const { return request_count_; }

private:
    int64_t request_count_ = 0;  // Intentionally not atomic for speed
};

```

```bash

# === tsan_suppressions.txt ===
# Known benign race in third-party logging library
race:spdlog::details::registry::instance

# Benign race in our approximate counters
race:PerformanceCounters::record_request

# Known issue in glibc (fixed in newer versions)
race:__cxa_guard_acquire

# Usage:
# TSAN_OPTIONS="suppressions=tsan_suppressions.txt" ./test_runner

```

---

## Notes

- **TSan must be in a separate build** — it's incompatible with ASan and MSan
- TSan detects races on C++ objects, **not** on raw hardware — it may miss races in inline assembly
- Stress tests should run for at least several seconds to exercise scheduling variations
- Use `std::this_thread::yield()` in stress tests to vary timing patterns
- `ctest --repeat until-fail:100` is a simple way to stress test non-deterministic code
- TSan reports are very precise: they show both conflicting accesses with full stack traces
- False positives in TSan are rare — investigate before suppressing
- Prefer `std::atomic` for shared counters over manual locking — less error-prone
