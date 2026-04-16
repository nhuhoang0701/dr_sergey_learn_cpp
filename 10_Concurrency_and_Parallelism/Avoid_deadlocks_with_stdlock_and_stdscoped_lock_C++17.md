# Avoid deadlocks with std::lock and std::scoped_lock (C++17)

**Category:** Concurrency & Parallelism  
**Item:** #94  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/thread/scoped_lock>  

---

## Topic Overview

A **deadlock** occurs when two or more threads each hold a lock the other needs, causing them to wait forever. The classic scenario is two threads acquiring the same two mutexes in opposite order. C++17's `std::scoped_lock` solves this by locking multiple mutexes atomically using a **deadlock avoidance algorithm**.

### The Deadlock Problem

```cpp

Thread 1:                Thread 2:
lock(mutex_A)            lock(mutex_B)      ← both succeed
lock(mutex_B) ← BLOCKS   lock(mutex_A) ← BLOCKS
...DEADLOCK...           ...DEADLOCK...

```

### Solutions

| Technique | Standard | Locks multiple? | RAII? | Notes |
| --- | --- | --- | --- | --- |
| Manual ordering | Any | Manual | No | Error-prone, doesn't scale |
| `std::lock(m1, m2, ...)` | C++11 | Yes | No (must adopt) | Low-level, need `lock_guard` wrapper |
| `std::scoped_lock(m1, m2)` | C++17 | Yes | Yes | **Recommended**: locks + RAII in one step |

### How std::scoped_lock Works

```cpp

#include <mutex>
#include <thread>
#include <iostream>

std::mutex m1, m2;
int account_a = 1000, account_b = 1000;

void transfer(int& from, int& to, std::mutex& from_m, std::mutex& to_m, int amount) {
    // scoped_lock locks BOTH mutexes atomically — no deadlock possible
    std::scoped_lock lock(from_m, to_m);
    if (from >= amount) {
        from -= amount;
        to += amount;
    }
    // Both mutexes released here (RAII)
}

int main() {
    // Even though threads lock in "opposite" order, no deadlock:
    std::thread t1([&]{ transfer(account_a, account_b, m1, m2, 100); });
    std::thread t2([&]{ transfer(account_b, account_a, m2, m1, 200); });

    t1.join();
    t2.join();
    std::cout << "A: " << account_a << ", B: " << account_b << "\n";
    // Total always = 2000 (no data race, no deadlock)
}

```

---

## Self-Assessment

### Q1: Show a deadlock with two mutexes acquired in opposite orders in two threads

**Answer:**

```cpp

#include <mutex>
#include <thread>
#include <iostream>
#include <chrono>

std::mutex mutex_a, mutex_b;
int shared_x = 0, shared_y = 0;

void thread_1() {
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Lock A first
    // Small sleep to make deadlock almost certain
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Then try B — BLOCKS!
    shared_x += shared_y;
}

void thread_2() {
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Lock B first — opposite order!
    std::this_thread::sleep_for(std::chrono::milliseconds(1));
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Then try A — BLOCKS!
    shared_y += shared_x;
}

int main() {
    std::thread t1(thread_1);
    std::thread t2(thread_2);
    // WARNING: This program will hang forever (deadlock)!
    //
    // t1 holds mutex_a, waits for mutex_b
    // t2 holds mutex_b, waits for mutex_a
    // Neither can proceed → deadlock
    //
    //   Thread 1:  [holds A] ──wants──→ B
    //   Thread 2:  [holds B] ──wants──→ A
    //   Circular dependency!

    t1.join();
    t2.join();
    std::cout << "This line is never reached\n";
}

```

**Explanation:** The deadlock occurs because Thread 1 locks `mutex_a` then tries `mutex_b`, while Thread 2 locks `mutex_b` then tries `mutex_a`. Each thread holds what the other needs, creating a circular wait. The `sleep_for` makes the race condition nearly deterministic — in real code, the timing is unpredictable, making deadlocks hard to reproduce.

### Q2: Fix the deadlock using std::scoped_lock with multiple mutexes

**Answer:**

```cpp

#include <mutex>
#include <thread>
#include <iostream>
#include <vector>

std::mutex mutex_a, mutex_b;
int shared_x = 100, shared_y = 200;

void thread_1() {
    // scoped_lock acquires BOTH mutexes without deadlock risk
    std::scoped_lock lock(mutex_a, mutex_b);
    shared_x += shared_y;
    std::cout << "Thread 1: x=" << shared_x << "\n";
    // Both mutexes released at end of scope
}

void thread_2() {
    // Even if listed in different order — still safe!
    std::scoped_lock lock(mutex_b, mutex_a);
    shared_y += shared_x;
    std::cout << "Thread 2: y=" << shared_y << "\n";
}

int main() {
    std::thread t1(thread_1);
    std::thread t2(thread_2);
    t1.join();
    t2.join();
    std::cout << "Final: x=" << shared_x << ", y=" << shared_y << "\n";
    // No deadlock! Output order depends on scheduling.

    // === Also works with more than 2 mutexes ===
    std::mutex m1, m2, m3;
    {
        std::scoped_lock lock(m1, m2, m3); // locks all 3 atomically
        // critical section
    } // all 3 released

    // === Pre-C++17 equivalent using std::lock + lock_guard ===
    // std::lock(mutex_a, mutex_b);  // lock both, deadlock-free
    // std::lock_guard<std::mutex> la(mutex_a, std::adopt_lock);
    // std::lock_guard<std::mutex> lb(mutex_b, std::adopt_lock);
    // // adopt_lock tells lock_guard: "already locked, just manage it"
}

```

**Explanation:** `std::scoped_lock` locks all mutexes atomically using an internal deadlock avoidance algorithm. The order of arguments doesn't matter — it will find a safe locking order. It also provides RAII: all mutexes are released when the `scoped_lock` goes out of scope, even if an exception is thrown.

### Q3: Explain how std::scoped_lock implements deadlock avoidance internally

**Answer:**

```cpp

#include <mutex>
#include <iostream>

// std::scoped_lock internally uses the same algorithm as std::lock().
// Here is a conceptual illustration of the "try-and-back-off" algorithm:

// Pseudocode of std::lock(m1, m2) internal algorithm:
//
// 1. Lock m1
// 2. try_lock m2
//    - If success: done ✓ (both locked)
//    - If fail:
//       a. Unlock m1 (release to prevent deadlock)
//       b. Lock m2
//       c. try_lock m1
//          - If success: done ✓ (both locked)
//          - If fail: unlock m2, goto step 1
//
// This avoids deadlock because:
// - No thread holds a lock while doing a BLOCKING acquire of another
// - try_lock is non-blocking — it either succeeds or returns immediately
// - If try_lock fails, the thread releases everything and retries
//
// This is the "try-lock, back-off" strategy (not lock ordering).

// Demonstration with try_lock to show the concept:
std::mutex m1, m2;

void lock_both_manually() {
    while (true) {
        m1.lock();
        if (m2.try_lock()) {
            return; // Both locked!
        }
        m1.unlock(); // Back off

        m2.lock();
        if (m1.try_lock()) {
            return; // Both locked!
        }
        m2.unlock(); // Back off
        // Retry...
    }
}

int main() {
    // In practice, just use scoped_lock:
    {
        std::scoped_lock lock(m1, m2);
        std::cout << "Both locked safely\n";
    }
    // Output: Both locked safely

    // KEY POINTS about the internal algorithm:
    // 1. It does NOT rely on a fixed global ordering of mutexes
    // 2. It uses try_lock to avoid blocking while holding other locks
    // 3. It backs off (releases locks) on failure to prevent livelock
    // 4. The algorithm is guaranteed to make progress (no starvation)
    // 5. Works with any number of mutexes (variadic template)

    // IMPORTANT: scoped_lock with a SINGLE mutex is equivalent to lock_guard:
    std::mutex m;
    {
        std::scoped_lock lock(m); // same as std::lock_guard<std::mutex> lock(m);
    }
}

```

**Explanation:** `std::scoped_lock` uses the same algorithm as `std::lock()`: it tries to lock all mutexes using a combination of `lock()` and `try_lock()`. If any `try_lock` fails, it releases all held locks and retries in a different order. This "try-and-back-off" approach guarantees no circular wait (the necessary condition for deadlock), without requiring a global mutex ordering convention.

---

## Notes

- **Always prefer `std::scoped_lock`** over manual lock ordering or `std::lock` + `adopt_lock` pairs.
- **Single-mutex case:** `std::scoped_lock(m)` is equivalent to `std::lock_guard(m)` — use either.
- **Exception safety:** `std::scoped_lock` releases all held mutexes in its destructor, even during stack unwinding.
- **`std::lock` (C++11):** The lower-level function that `scoped_lock` wraps. Use it when you need to separate locking from scope management.
- **Lock hierarchy:** For complex systems with many mutexes, consider a lock hierarchy (assigning levels to mutexes) in addition to `scoped_lock`.
- Test with `-fsanitize=thread` to detect data races and lock-order issues.

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
