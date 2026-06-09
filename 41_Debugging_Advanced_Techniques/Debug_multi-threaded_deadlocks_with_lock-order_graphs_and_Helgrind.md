# Debug multi-threaded deadlocks with lock-order graphs and Helgrind

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://valgrind.org/docs/manual/hg-manual.html>  

---

## Topic Overview

A deadlock is one of the most frustrating bugs to reproduce and diagnose. Your program simply stops responding, and there's no crash, no error message, and no obvious indication of what went wrong. The root cause is almost always the same: two or more threads each holding a lock the other needs.

The good news is that the pattern is well-understood, and specialized tools can detect the potential for a deadlock even before it actually manifests. They do this by building a lock-order graph at runtime - a record of which locks have been acquired in which order - and looking for cycles in that graph.

### Classic Deadlock Pattern

The simplest possible deadlock involves two mutexes and two threads, each locking them in the opposite order. This is the canonical example:

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
    std::lock_guard lock_a(mutex_a);  // Then lock A - DEADLOCK!
}

// Fix: consistent lock ordering or std::scoped_lock
void thread1_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // Locks both atomically
}
```

Thread 1 locks A then B. Thread 2 locks B then A. If thread 1 acquires A while thread 2 acquires B simultaneously, each thread is now waiting for the other to release a lock it will never release. The program hangs forever.

The fix in `thread1_fixed` works because `std::scoped_lock` uses a deadlock-avoidance algorithm internally - it doesn't matter which order you list the mutexes, it will always acquire them in a safe sequence.

### Using Helgrind

Helgrind's value is that it catches the problematic lock ordering pattern without you having to actually hit the deadlock. In testing, the two threads might happen to run in an order that never triggers the deadlock, so you ship the code and it hangs in production six months later. Helgrind builds the ordering graph from every run and flags the inconsistency immediately:

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

The output tells you exactly which two source locations established the conflicting lock orders. Even if the deadlock never triggered during your test run, Helgrind is telling you it could, and exactly where to look.

### ThreadSanitizer (TSan) - Faster Alternative

If you can recompile your program, TSan is usually the better choice. It works by inserting instrumentation at compile time, which is much lighter than Valgrind's dynamic binary translation. The output is similar in quality but the runtime cost is dramatically lower:

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

The 5-10x overhead means you can often run TSan-instrumented binaries in a CI pipeline or even in a load test environment without them being completely unusable. Helgrind's 20-100x overhead usually makes that impractical.

### Lock-Order Graph Approach

If you want to roll your own lightweight lock-order validator for debug builds, the core idea is straightforward: maintain a graph where an edge from A to B means "a thread acquired A while already holding B." At each lock acquisition, check whether adding the new edge would create a cycle. If yes, you have a potential deadlock. The implementation below does exactly this:

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

This is a simplified version of what Helgrind does. It fires an assert the first time a lock-order violation is detected, giving you a stack trace at the exact point of inversion rather than waiting for a deadlock to materialize.

---

## Self-Assessment

### Q1: How does lock-order graph cycle detection work

You build a directed graph at runtime where each edge A -> B means "some thread locked A then locked B while still holding A." A cycle in this graph (A -> B -> A) means that one execution path locks A then B, and another locks B then A. Even if those two paths have never executed concurrently in your test runs, the cycle proves that a deadlock is possible. The graph is constructed by intercepting every lock and unlock call - tools like Helgrind do this via dynamic binary instrumentation, while TSan does it via compile-time hooks.

### Q2: What's the difference between Helgrind and TSan

Helgrind uses Valgrind's dynamic binary instrumentation: it translates your compiled binary at runtime, injecting checks for every memory access and synchronization operation. This works on any binary without recompilation, but adds significant overhead (20-100x). TSan uses compile-time instrumentation inserted by the compiler when you pass `-fsanitize=thread`. Because the instrumentation is generated directly into your code at build time, it's much faster (5-10x). TSan also detects data races more precisely and is the preferred choice when you can recompile.

### Q3: How does `std::scoped_lock` prevent deadlocks

`std::scoped_lock` calls `std::lock()` internally, which implements a deadlock-avoidance algorithm based on try-and-back-off. When acquiring multiple mutexes, `std::lock` tries to lock all of them without creating a cycle in the wait graph. If it can't acquire one of the mutexes without blocking, it releases the ones it already holds and tries again in a different order. This is why `std::scoped_lock` is safe to use even when different threads specify the mutexes in different orders - the algorithm handles it.

---

## Notes

- TSan and Helgrind catch lock-order violations even when the deadlock doesn't manifest in testing.
- Always use `std::scoped_lock` for multi-lock acquisition.
- `-fsanitize=thread` is incompatible with `-fsanitize=address` - run them separately.
- Clang's TSan has better C++ support than GCC's.
