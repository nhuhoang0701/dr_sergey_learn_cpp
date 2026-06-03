# Implement sequence locks (seqlock) for read-heavy concurrent data

**Category:** Concurrency and Parallelism  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Seqlock>  

---

## Topic Overview

A **seqlock** is a beautifully simple lock design that shines when you have a lot more readers than writers. The key idea is that readers are completely lock-free - they never block a writer, and they never spin waiting for a lock to become available. Instead, readers are *optimistic*: they grab the data, then check afterward whether a write happened during the read. If it did, they just try again.

The mechanism is a single atomic sequence counter. An even value means the data is stable. An odd value means a write is in progress. Readers snapshot the counter at the start, copy the data, then check whether the counter is still the same even value. Writers bump the counter to odd to signal "I'm writing," update the data, then bump it back to even to signal "I'm done."

Here is what a complete seqlock looks like in practice:

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

Notice that `read_begin()` spins only if a write is actively happening (odd counter). In the common case - no concurrent writer - it falls straight through with a single acquire load. The retry loop in `read_data()` handles the rare collision. If no write happened while you were reading, `read_retry()` returns false and you're done in one pass.

---

## Self-Assessment

### Q1: When is a seqlock better than a mutex or rwlock

When reads vastly outnumber writes (a ratio above 100:1 is a good rule of thumb), and when reads must never block waiting for a lock. The seqlock also works particularly well when the protected data is small - ideally something that fits in a handful of cache lines. The Linux kernel uses seqlocks for reading timestamp and clock data, which is an almost perfect fit: the timer interrupt updates the clock (rare write), and millions of system calls read it (constant reads).

### Q2: What are the limitations

Readers may have to retry if a write happens during their read. Because of this, the protected data must be trivially copyable - there can be no internal pointers or resources that would be broken by copying a partially-updated value. Readers can also starve if writes arrive in a continuous stream with no gap between them, because the counter never settles to an even value long enough for a reader to finish. This makes seqlocks unsuitable for large or complex data structures.

### Q3: Why is seqlock used for gettimeofday()

The system clock is updated by a timer interrupt, which is the rare writer in this picture. Every user-space call to read the time is a reader. A seqlock lets each reader pay almost nothing - it loads the counter, copies the timestamp, and checks the counter again. There is no cache-line bouncing on a shared lock variable in the common case, which is exactly what you want when you have millions of readers per second.

---

## Notes

- Linux uses seqlocks for timekeeping, network statistics, and VFS metadata - all classic read-heavy scenarios.
- Readers are effectively wait-free in the uncontended case (a single load plus a comparison).
- Writers still need mutual exclusion with each other - combine the seqlock with a separate mutex for write-write exclusion.
- Works best with small, trivially-copyable data like timestamps, coordinates, and counters.
