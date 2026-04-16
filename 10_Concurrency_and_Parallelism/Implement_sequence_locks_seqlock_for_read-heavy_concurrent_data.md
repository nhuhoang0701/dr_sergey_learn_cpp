# Implement sequence locks (seqlock) for read-heavy concurrent data

**Category:** Concurrency and Parallelism  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Seqlock>  

---

## Topic Overview

A **seqlock** allows lock-free reads with very low overhead. Readers don't block writers. Reads are optimistic: they check if the data was modified during the read.

### Implementation

```cpp

#include <atomic>
#include <thread>

class SeqLock {
    std::atomic<unsigned> seq_{0};  // Even = stable, Odd = write in progress

public:
    unsigned read_begin() const {
        unsigned s;
        do {
            s = seq_.load(std::memory_order_acquire);
        } while (s & 1);  // Wait while write in progress (odd)
        return s;
    }

    bool read_retry(unsigned start_seq) const {
        std::atomic_thread_fence(std::memory_order_acquire);
        return seq_.load(std::memory_order_relaxed) != start_seq;
    }

    void write_lock() {
        // Increment to odd (signals write in progress)
        seq_.fetch_add(1, std::memory_order_release);
    }

    void write_unlock() {
        // Increment to even (signals write complete)
        seq_.fetch_add(1, std::memory_order_release);
    }
};

// Usage:
struct SharedData {
    double x, y, z;
};

SeqLock lock;
SharedData data{1.0, 2.0, 3.0};

// Reader (lock-free, wait-free on fast path):
SharedData read_data() {
    SharedData local;
    unsigned seq;
    do {
        seq = lock.read_begin();
        local = data;  // Copy data (may be torn)
    } while (lock.read_retry(seq));  // Retry if write happened during read
    return local;
}

// Writer (mutual exclusion among writers required separately):
void write_data(double nx, double ny, double nz) {
    lock.write_lock();
    data = {nx, ny, nz};  // Update data
    lock.write_unlock();
}

```

---

## Self-Assessment

### Q1: When is a seqlock better than a mutex or rwlock

When reads vastly outnumber writes (>100:1 ratio), when reads must never block, and when the protected data is small (fits in a few cache lines). Seqlocks are used in the Linux kernel for timestamp reading.

### Q2: What are the limitations

Readers may need to retry if a write happens during the read. The protected data must be trivially copyable (no pointers to internal data). Readers can starve if writes are continuous. Not suitable for large data structures.

### Q3: Why is seqlock used for gettimeofday()

The system clock is updated by a timer interrupt (rare writer) and read by millions of user-space calls (frequent readers). A seqlock allows reads to be nearly free — no cache line bouncing on the lock in the common case.

---

## Notes

- Linux kernel uses seqlocks for timekeeping, network statistics, and VFS metadata.
- Readers are wait-free in the uncontended case (single load + comparison).
- Combine with a separate mutex for write-write exclusion.
- Works best with small, trivially-copyable data (timestamps, coordinates, counters).
