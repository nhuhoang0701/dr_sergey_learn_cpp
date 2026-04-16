# Design thread-safe classes with minimal locking granularity

**Category:** OOP Design

---

## Topic Overview

| Strategy | Granularity | Throughput | Complexity |
| --- | :---: | :---: | :---: |
| Single mutex (coarse) | Whole object | Low | **Simple** |
| Per-member mutex (fine) | Individual fields | **High** | High |
| Read-write lock | Read vs write | Medium-High | Medium |
| Lock-free atomics | Operation-level | **Highest** | Very high |
| Immutable + swap | Whole object | High | Medium |

### Design Principles

```cpp

1. Hold locks for the shortest possible time
2. Never call unknown code (callbacks, virtual functions) while holding a lock
3. Lock ordering: always acquire locks in a consistent order to prevent deadlocks
4. Prefer shared_mutex for read-heavy workloads
5. Copy data out under lock, process outside

```

---

## Self-Assessment

### Q1: Compare coarse vs fine-grained locking

**Answer:**

```cpp

#include <mutex>
#include <shared_mutex>
#include <string>
#include <unordered_map>
#include <vector>
#include <iostream>

// COARSE: single lock, simple but blocks everything
class CoarseUserStore {
    mutable std::mutex mtx_;
    std::unordered_map<int, std::string> users_;
public:
    void add(int id, std::string name) {
        std::lock_guard lk(mtx_);
        users_[id] = std::move(name);
    }
    std::string get(int id) const {
        std::lock_guard lk(mtx_);
        auto it = users_.find(id);
        return it != users_.end() ? it->second : "";
    }
    // Problem: get() blocks add() and vice versa
};

// FINE: read-write lock allows concurrent reads
class FineUserStore {
    mutable std::shared_mutex mtx_;
    std::unordered_map<int, std::string> users_;
public:
    void add(int id, std::string name) {
        std::unique_lock lk(mtx_);  // Exclusive for writes
        users_[id] = std::move(name);
    }
    std::string get(int id) const {
        std::shared_lock lk(mtx_);  // Shared for reads
        auto it = users_.find(id);
        return it != users_.end() ? it->second : "";
    }
    // Multiple get() calls run in parallel!
};

// FINEST: per-bucket locking for maps
class StripedMap {
    static constexpr size_t NUM_STRIPES = 16;
    struct Stripe {
        mutable std::mutex mtx;
        std::unordered_map<int, std::string> data;
    };
    std::array<Stripe, NUM_STRIPES> stripes_;

    Stripe& stripe_for(int key) {
        return stripes_[std::hash<int>{}(key) % NUM_STRIPES];
    }
public:
    void put(int key, std::string value) {
        auto& s = stripe_for(key);
        std::lock_guard lk(s.mtx);
        s.data[key] = std::move(value);
    }
    std::string get(int key) {
        auto& s = stripe_for(key);
        std::lock_guard lk(s.mtx);
        auto it = s.data.find(key);
        return it != s.data.end() ? it->second : "";
    }
    // Different stripes accessed concurrently!
};

```

### Q2: Show the "copy out, process outside" pattern

**Answer:**

```cpp

#include <mutex>
#include <vector>
#include <functional>
#include <iostream>

class EventProcessor {
    std::mutex mtx_;
    std::vector<std::function<void()>> pending_;
    std::vector<std::function<void()>> handlers_;
public:
    void add_handler(std::function<void()> h) {
        std::lock_guard lk(mtx_);
        handlers_.push_back(std::move(h));
    }

    void enqueue(std::function<void()> event) {
        std::lock_guard lk(mtx_);
        pending_.push_back(std::move(event));
    }

    void process() {
        // Step 1: COPY data out under lock
        std::vector<std::function<void()>> to_process;
        std::vector<std::function<void()>> handlers;
        {
            std::lock_guard lk(mtx_);
            to_process = std::move(pending_);
            pending_.clear();
            handlers = handlers_;  // Copy, don't move
        }
        // Step 2: Process OUTSIDE lock!
        // Handlers may call add_handler/enqueue without deadlock
        for (auto& event : to_process) {
            event();
            for (auto& h : handlers) h();
        }
    }
};

```

### Q3: Implement a lock-free concurrent counter and compare with mutex

**Answer:**

```cpp

#include <atomic>
#include <mutex>
#include <thread>
#include <vector>
#include <chrono>
#include <iostream>

// Lock-based counter
class MutexCounter {
    std::mutex mtx_;
    int64_t count_ = 0;
public:
    void increment() {
        std::lock_guard lk(mtx_);
        ++count_;
    }
    int64_t get() const {
        return count_;  // Reads are atomic on most platforms for int64
    }
};

// Lock-free counter
class AtomicCounter {
    std::atomic<int64_t> count_{0};
public:
    void increment() {
        count_.fetch_add(1, std::memory_order_relaxed);
    }
    int64_t get() const {
        return count_.load(std::memory_order_relaxed);
    }
};

// Lock-free with per-thread sharding (highest throughput)
class ShardedCounter {
    struct alignas(64) Shard {  // Cache-line aligned!
        std::atomic<int64_t> count{0};
    };
    std::array<Shard, 16> shards_;

    static size_t shard_index() {
        static std::atomic<size_t> next{0};
        thread_local size_t idx = next.fetch_add(1) % 16;
        return idx;
    }
public:
    void increment() {
        shards_[shard_index()].count.fetch_add(1, std::memory_order_relaxed);
    }
    int64_t get() const {
        int64_t total = 0;
        for (auto& s : shards_)
            total += s.count.load(std::memory_order_relaxed);
        return total;
    }
};

template<typename Counter>
long long bench(const char* name, int num_threads, int ops_per_thread) {
    Counter c;
    auto start = std::chrono::high_resolution_clock::now();
    std::vector<std::thread> threads;
    for (int t = 0; t < num_threads; ++t)
        threads.emplace_back([&] {
            for (int i = 0; i < ops_per_thread; ++i) c.increment();
        });
    for (auto& t : threads) t.join();
    auto end = std::chrono::high_resolution_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(end - start).count();
    std::cout << name << ": " << ms << "ms (count=" << c.get() << ")\n";
    return ms;
}

int main() {
    constexpr int T = 8, N = 1'000'000;
    bench<MutexCounter>("Mutex", T, N);
    bench<AtomicCounter>("Atomic", T, N);
    bench<ShardedCounter>("Sharded", T, N);
    return 0;
}

```

---

## Notes

- **Rule of thumb:** start with `shared_mutex`, optimize to finer granularity only when profiling shows contention
- Never hold a lock while calling user callbacks or virtual functions — deadlock risk
- `alignas(64)` prevents false sharing between cache lines in sharded data structures
- Copy snapshots out under lock, process outside — minimizes lock hold time
- Lock-free code is **not always faster** — cache coherency traffic can dominate
- Use ThreadSanitizer (`-fsanitize=thread`) to detect data races during testing
