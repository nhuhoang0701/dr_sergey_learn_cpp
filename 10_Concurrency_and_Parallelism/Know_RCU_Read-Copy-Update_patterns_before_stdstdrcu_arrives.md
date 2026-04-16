# Know RCU (Read-Copy-Update) patterns before std::rcu arrives

**Category:** Concurrency and Parallelism  
**Standard:** C++26 (proposed)  
**Reference:** <https://wg21.link/P2545>  

---

## Topic Overview

RCU provides extremely fast reader access (zero overhead reads) at the cost of delayed reclamation. Writers create new versions; old versions are freed once all readers are done.

### Simplified RCU Pattern

```cpp

#include <atomic>
#include <memory>
#include <thread>

// RCU-protected data: readers see a consistent snapshot
class RcuConfig {
    struct Config {
        std::string hostname;
        int port;
        int max_connections;
    };

    std::atomic<Config*> current_;

public:
    RcuConfig(Config* initial) : current_(initial) {}

    // Read side: just load the pointer (extremely fast)
    const Config* read() const {
        return current_.load(std::memory_order_acquire);
        // Caller must not hold this pointer across a "quiescent state"
    }

    // Write side: create new version, publish, reclaim old
    void update(Config* new_config) {
        Config* old = current_.exchange(new_config, std::memory_order_release);
        // Must wait for all readers holding old pointer to finish
        synchronize_rcu();
        delete old;
    }

private:
    void synchronize_rcu() {
        // Simplified: in production, use epoch-based or quiescent-state reclamation
        // For now, just sleep to let readers finish
        std::this_thread::sleep_for(std::chrono::milliseconds(10));
    }
};

```

### Epoch-Based Reclamation

```cpp

#include <atomic>
#include <vector>
#include <functional>

class EpochReclaimer {
    static constexpr int NUM_EPOCHS = 3;
    std::atomic<uint64_t> global_epoch_{0};
    // Per-thread local epoch tracking
    static thread_local uint64_t local_epoch_;
    static thread_local bool in_critical_;

    std::vector<std::function<void()>> retire_lists_[NUM_EPOCHS];

public:
    void enter_critical() {
        in_critical_ = true;
        local_epoch_ = global_epoch_.load(std::memory_order_acquire);
    }

    void exit_critical() {
        in_critical_ = false;
    }

    void retire(std::function<void()> cleanup) {
        auto epoch = global_epoch_.load(std::memory_order_relaxed);
        retire_lists_[epoch % NUM_EPOCHS].push_back(std::move(cleanup));
    }

    void try_reclaim() {
        auto new_epoch = global_epoch_.load() + 1;
        // If all threads have passed the current epoch...
        global_epoch_.store(new_epoch, std::memory_order_release);
        // Safe to reclaim (epoch - 2) nodes
        auto& list = retire_lists_[(new_epoch - 2) % NUM_EPOCHS];
        for (auto& fn : list) fn();
        list.clear();
    }
};

```

---

## Self-Assessment

### Q1: Why is RCU faster than rwlock for readers

RCU readers don't modify any shared state (no lock counter to increment). This means zero cache-line contention, zero atomic operations on the read path. An rwlock requires an atomic increment/decrement of the reader count, causing cache-line bouncing.

### Q2: What is the trade-off

Writers must wait for all readers to finish before reclaiming old data. This means memory usage grows during high-write periods (old versions accumulate). RCU is best for read-dominated workloads (routing tables, configuration).

### Q3: What will std::rcu provide

P2545 proposes: `std::rcu_reader` (RAII read-side critical section), `std::rcu_synchronize()` (wait for readers), and `std::rcu_retire(ptr)` (deferred deletion). This standardizes what the Linux kernel has used since 2002.

---

## Notes

- Linux kernel uses RCU for routing tables, filesystems (dcache), and module unloading.
- Userspace RCU library: liburcu provides production RCU for C/C++.
- RCU + hazard pointers are complementary lock-free memory reclamation schemes.
- `std::rcu` targets C++26 — use liburcu or custom implementation until then.
