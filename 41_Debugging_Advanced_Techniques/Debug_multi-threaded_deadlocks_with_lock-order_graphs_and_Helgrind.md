# Debug multi-threaded deadlocks with lock-order graphs and Helgrind

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://valgrind.org/docs/manual/hg-manual.html>  

---

## Topic Overview

Deadlocks occur when threads hold locks in different orders. Detection tools build lock-order graphs to identify cycles.

### Classic Deadlock Pattern

```cpp

#include <mutex>
#include <thread>

std::mutex mutex_a, mutex_b;

void thread1() {
    std::lock_guard lock_a(mutex_a);  // Lock A first
    std::lock_guard lock_b(mutex_b);  // Then lock B
}

void thread2() {
    std::lock_guard lock_b(mutex_b);  // Lock B first
    std::lock_guard lock_a(mutex_a);  // Then lock A — DEADLOCK!
}

// Fix: consistent lock ordering or std::scoped_lock
void thread1_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // Locks both atomically
}

```

### Using Helgrind

```bash

# Helgrind detects lock-order violations even WITHOUT a deadlock occurring
valgrind --tool=helgrind ./my_program

# Output:
# Thread #2: lock order "0x601040 before 0x601080" violated
# ...previously established order was:
#   first  0x601040 (mutex_a) at thread1 (main.cpp:5)
#   then   0x601080 (mutex_b) at thread1 (main.cpp:6)
# ...now acquiring:
#   first  0x601080 (mutex_b) at thread2 (main.cpp:10)
#   then   0x601040 (mutex_a) at thread2 (main.cpp:11)

```

### ThreadSanitizer (TSan) — Faster Alternative

```bash

# Compile with TSan
g++ -fsanitize=thread -g -O1 main.cpp -o main
./main

# TSan detects:
# - Data races
# - Lock-order inversions
# - Use of destroyed mutex
# - Signal-unsafe lock operations

# TSan is ~5-10x overhead (vs Helgrind's ~20-100x)

```

### Lock-Order Graph Approach

```cpp

#include <mutex>
#include <map>
#include <set>
#include <thread>
#include <cassert>

// Simple lock-order validator (debug builds only)
#ifdef DEBUG
class LockOrderChecker {
    static thread_local std::set<const void*> held_locks_;
    static inline std::mutex graph_mutex_;
    static inline std::map<const void*, std::set<const void*>> order_graph_;
public:
    static void on_lock(const void* mtx) {
        std::lock_guard g(graph_mutex_);
        // Every currently held lock must always be acquired before mtx
        for (auto* held : held_locks_) {
            order_graph_[held].insert(mtx);
            // Check: mtx should NOT already come before any held lock
            assert(order_graph_[mtx].count(held) == 0 &&
                   "Lock-order inversion detected!");
        }
        held_locks_.insert(mtx);
    }
    static void on_unlock(const void* mtx) {
        held_locks_.erase(mtx);
    }
};
thread_local std::set<const void*> LockOrderChecker::held_locks_;
#endif

```

---

## Self-Assessment

### Q1: How does lock-order graph cycle detection work

Build a directed graph where an edge A→B means "thread locked A then locked B while holding A." A cycle in this graph (A→B→A) means a deadlock is possible. The graph is built at runtime by intercepting lock/unlock calls. Any cycle — even if the deadlock hasn't occurred yet — is a potential deadlock.

### Q2: What's the difference between Helgrind and TSan

Helgrind uses Valgrind's dynamic binary instrumentation (slower, no recompilation). TSan uses compile-time instrumentation (faster, requires `-fsanitize=thread`). TSan also detects data races more precisely. TSan is preferred when you can recompile.

### Q3: How does `std::scoped_lock` prevent deadlocks

`std::scoped_lock` uses `std::lock()` internally, which implements a deadlock-avoidance algorithm (try-and-back-off). It atomically acquires all mutexes regardless of the order they're specified, preventing lock-order inversions.

---

## Notes

- TSan and Helgrind catch lock-order violations even when the deadlock doesn't manifest in testing.
- Always use `std::scoped_lock` for multi-lock acquisition.
- `-fsanitize=thread` is incompatible with `-fsanitize=address` — run them separately.
- Clang's TSan has better C++ support than GCC's.
