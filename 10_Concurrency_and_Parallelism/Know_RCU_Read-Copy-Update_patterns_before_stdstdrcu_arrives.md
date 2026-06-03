# Know RCU (Read-Copy-Update) patterns before std::rcu arrives

**Category:** Concurrency and Parallelism  
**Standard:** C++26 (proposed)  
**Reference:** <https://wg21.link/P2545>  

---

## Topic Overview

RCU is built around one core trade-off: readers pay essentially nothing, and writers pay by having to wait before they can free old data. The name "Read-Copy-Update" describes the writer's protocol exactly - you read the old version, create a copy with your changes, atomically publish the new version, and then wait until all threads that might still be reading the old version have moved on. Only then do you free the old one.

The result is that every read path is just an atomic pointer load - no lock counter to increment, no cache-line bouncing, no blocking. This makes RCU reads faster than even a read-write lock in highly contended scenarios.

### Simplified RCU Pattern

Here is a minimal RCU-style configuration manager. Notice how the read path is a single line: just load the pointer:

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

The sleep in `synchronize_rcu()` is a placeholder for a real grace-period mechanism. In production code you would use epoch-based reclamation or quiescent-state tracking to know precisely when all old readers have finished, rather than guessing with a timer.

### Epoch-Based Reclamation

The real challenge in RCU is knowing when it is safe to free the old data. Here is a sketch of an epoch-based approach, where the global epoch counter tracks how far all threads have progressed:

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

The key intuition here is the three-epoch cycle. A node retired in epoch N can only be freed once the global epoch has advanced past N+1, because that guarantees every thread that could have been reading in epoch N has since exited its critical section and re-entered in a newer epoch.

---

## Self-Assessment

### Q1: Why is RCU faster than rwlock for readers

RCU readers don't touch any shared state at all - no lock counter to increment, no atomic operation, no chance of a cache-line bouncing between cores. An rwlock requires an atomic increment when entering the read lock and an atomic decrement when releasing it, and every atomic write to that shared counter forces other cores to do cache-coherence work. Under heavy read concurrency, that coherence traffic becomes a serious bottleneck. RCU sidesteps it entirely.

### Q2: What is the trade-off

Writers must wait for all currently-active readers to finish before they can free the old version. During a period of heavy writes, old versions accumulate in memory - the retire list keeps growing until the grace period passes. RCU is the right tool for read-dominated workloads like routing tables and configuration data, but it can be a poor fit for write-heavy scenarios where memory consumption during grace periods is a concern.

### Q3: What will std::rcu provide

The proposal P2545 standardizes three things: `std::rcu_reader` as an RAII type for entering and exiting a read-side critical section; `std::rcu_synchronize()` for blocking the writer until all current readers have finished; and `std::rcu_retire(ptr)` for scheduling deferred deletion once the next grace period completes. This brings to standard C++ a mechanism that the Linux kernel has been using effectively since 2002.

---

## Notes

- The Linux kernel uses RCU for routing tables, the dcache in the filesystem, and module unloading - all places where reads must be blazing fast.
- The userspace RCU library `liburcu` provides production-grade RCU for C and C++ today if you cannot wait for C++26.
- RCU and hazard pointers are complementary tools, not competing ones - they have different trade-offs and are often used together in the same codebase.
- `std::rcu` targets C++26, so for now use `liburcu` or a custom implementation.
