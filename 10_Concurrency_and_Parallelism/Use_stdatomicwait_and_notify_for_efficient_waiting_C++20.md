# Use std::atomic::wait and notify for efficient waiting (C++20)

**Category:** Concurrency & Parallelism  
**Item:** #484  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/wait>  

---

## Topic Overview

C++20 adds `wait()`, `notify_one()`, and `notify_all()` to `std::atomic`. These provide efficient blocking similar to `condition_variable` but directly on atomic variables — no mutex required.

### API

```cpp

std::atomic<int> flag{0};

// WAITER: blocks until flag != old_value
flag.wait(old_value);   // blocks if flag == old_value
                         // wakes when flag might have changed

// NOTIFIER: wakes blocked waiters
flag.store(1);
flag.notify_one();       // wake one waiter
flag.notify_all();       // wake all waiters

```

### Comparison: Spin vs Wait vs Condition Variable

```cpp

Spin loop:               atomic::wait:            condition_variable:
while(flag == 0) {}      flag.wait(0);            cv.wait(lock, pred);
                         
CPU: 100% burn           CPU: sleeps              CPU: sleeps
Latency: ~ns             Latency: ~μs             Latency: ~μs
Power: maximum            Power: minimal           Power: minimal
Needs: nothing           Needs: nothing           Needs: mutex + cv

```

---

## Self-Assessment

### Q1: Replace a spin-wait on an atomic bool with atomic::wait for energy-efficient blocking

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <iostream>
#include <chrono>

// === SPIN WAIT (wastes CPU cycles) ===
void demo_spin_wait() {
    std::atomic<bool> ready{false};

    std::thread worker([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        ready.store(true, std::memory_order_release);
        // No notification needed — spin loop polls continuously
    });

    // Burns CPU cycles checking ready in a tight loop
    while (!ready.load(std::memory_order_acquire))
        ; // CPU: 100% on this core

    std::cout << "Spin: ready!\n";
    worker.join();
}

// === ATOMIC WAIT (energy-efficient) ===
void demo_atomic_wait() {
    std::atomic<bool> ready{false};

    std::thread worker([&] {
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        ready.store(true, std::memory_order_release);
        ready.notify_one(); // MUST notify after store!
    });

    // Blocks efficiently — OS puts thread to sleep
    ready.wait(false); // blocks while ready == false
    //         ^^^^^
    // "wait while the value equals 'false'"
    // wakes when notify is called AND ready != false

    std::cout << "Wait: ready!\n";
    worker.join();
}

int main() {
    demo_spin_wait();
    demo_atomic_wait();

    // Both produce same output:
    // Spin: ready!
    // Wait: ready!
    //
    // But atomic::wait uses ~0% CPU while waiting,
    // while spin uses 100% CPU on one core.
    //
    // Platform implementation:
    //   Linux: uses futex() syscall (very efficient)
    //   Windows: WaitOnAddress() / WakeByAddressSingle()
    //   macOS: __ulock_wait() / __ulock_wake()
}

```

**Explanation:** `atomic::wait(old_value)` blocks the thread if the atomic's current value equals `old_value`. The OS wakes the thread when `notify_one()`/`notify_all()` is called. Unlike a spin loop, the waiting thread consumes zero CPU cycles. Unlike `condition_variable`, no mutex is needed.

### Q2: Show that notify_one wakes one waiter and notify_all wakes all waiters

**Answer:**

```cpp

#include <atomic>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>

int main() {
    // === notify_one: wakes exactly ONE waiter ===
    {
        std::atomic<int> gate{0};
        std::atomic<int> woken{0};

        auto waiter = [&](int id) {
            gate.wait(0); // block while gate == 0
            woken.fetch_add(1, std::memory_order_relaxed);
            std::cout << "  Waiter " << id << " woke up\n";
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i)
            threads.emplace_back(waiter, i);

        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "=== notify_one ===\n";
        gate.store(1);
        gate.notify_one(); // wake ONE waiter

        std::this_thread::sleep_for(std::chrono::milliseconds(100));
        std::cout << "Woken after notify_one: " << woken.load() << "\n";

        // Wake the rest so threads can be joined
        gate.notify_all();
        for (auto& t : threads) t.join();
        std::cout << "Total woken: " << woken.load() << "\n\n";
    }

    // === notify_all: wakes ALL waiters ===
    {
        std::atomic<int> gate{0};
        std::atomic<int> woken{0};

        auto waiter = [&](int id) {
            gate.wait(0);
            woken.fetch_add(1, std::memory_order_relaxed);
            std::cout << "  Waiter " << id << " woke up\n";
        };

        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i)
            threads.emplace_back(waiter, i);

        std::this_thread::sleep_for(std::chrono::milliseconds(50));
        std::cout << "=== notify_all ===\n";
        gate.store(1);
        gate.notify_all(); // wake ALL waiters at once

        for (auto& t : threads) t.join();
        std::cout << "Total woken: " << woken.load() << "\n";
    }

    // Output:
    // === notify_one ===
    //   Waiter 2 woke up
    // Woken after notify_one: 1
    //   Waiter 0 woke up
    //   Waiter 1 woke up
    //   Waiter 3 woke up
    // Total woken: 4
    //
    // === notify_all ===
    //   Waiter 0 woke up
    //   Waiter 3 woke up
    //   Waiter 1 woke up
    //   Waiter 2 woke up
    // Total woken: 4
}

```

**Explanation:** `notify_one()` wakes exactly one waiting thread (the kernel chooses which). `notify_all()` wakes all threads blocked on `wait()` for that atomic. After waking, each thread re-checks the value — if it still matches `old_value`, it goes back to sleep (spurious wakeup protection is built in).

### Q3: Compare atomic wait vs condition_variable for latency and power consumption

**Answer:**

```cpp

═══════════════════════════════════════════════════════════
COMPARISON: atomic::wait vs condition_variable vs spin
═══════════════════════════════════════════════════════════

┌───────────────────┬──────────────┬──────────────────┬──────────────┐
│                   │ Spin loop    │ atomic::wait     │ cond_variable│
├───────────────────┼──────────────┼──────────────────┼──────────────┤
│ CPU usage (wait)  │ 100% one core│ ~0%              │ ~0%          │
│ Wake latency      │ ~10-50 ns    │ ~1-5 μs          │ ~2-10 μs     │
│ Requires mutex    │ No           │ No               │ Yes          │
│ Spurious wakeups  │ N/A          │ Handled internally│ Must handle  │
│ Predicate check   │ Manual       │ Built-in (value) │ Lambda       │
│ Complex condition │ Yes          │ No (value only)  │ Yes          │
│ Multiple waiters  │ Trivial      │ notify_one/all   │ notify_one/all│
│ Mutex scope data  │ No           │ No               │ Yes          │
│ Standard          │ C++11        │ C++20            │ C++11        │
└───────────────────┴──────────────┴──────────────────┴──────────────┘

WHEN TO USE WHICH:
──────────────────
atomic::wait:
  ✓ Simple flag/counter-based synchronization
  ✓ No additional data to protect with a mutex
  ✓ Low power consumption with moderate latency
  ✓ Simpler code than condition_variable
  
condition_variable:
  ✓ Complex predicates ("queue not empty AND not paused")
  ✓ Need to protect shared state with the same lock
  ✓ Already using mutex for data access
  ✓ condition_variable_any + stop_token (C++20)

Spin loop:
  ✓ Ultra-low latency required (< μs)
  ✓ Wait time is very short (< 100 ns expected)
  ✓ Dedicated CPU core available (real-time systems)
  ✗ NEVER use for general-purpose waiting

```

```cpp

#include <atomic>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <iostream>
#include <chrono>

// Benchmark: measure wake-up latency
int main() {
    constexpr int TRIALS = 100'000;

    // === atomic::wait latency ===
    {
        std::atomic<int> flag{0};
        auto start = std::chrono::steady_clock::now();

        std::thread worker([&] {
            for (int i = 0; i < TRIALS; ++i) {
                flag.wait(0);
                flag.store(0);
                flag.notify_one();
            }
        });

        for (int i = 0; i < TRIALS; ++i) {
            flag.store(1);
            flag.notify_one();
            flag.wait(1);
        }
        worker.join();

        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "atomic::wait round-trip: " << ns / TRIALS << " ns\n";
    }

    // === condition_variable latency ===
    {
        std::mutex mtx;
        std::condition_variable cv;
        bool ready = false;
        auto start = std::chrono::steady_clock::now();

        std::thread worker([&] {
            for (int i = 0; i < TRIALS; ++i) {
                std::unique_lock lock(mtx);
                cv.wait(lock, [&] { return ready; });
                ready = false;
                cv.notify_one();
            }
        });

        for (int i = 0; i < TRIALS; ++i) {
            {
                std::lock_guard lock(mtx);
                ready = true;
            }
            cv.notify_one();
            std::unique_lock lock(mtx);
            cv.wait(lock, [&] { return !ready; });
        }
        worker.join();

        auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << "cond_var round-trip:     " << ns / TRIALS << " ns\n";
    }

    // Typical output:
    // atomic::wait round-trip: 1500 ns
    // cond_var round-trip:     3000 ns
    // atomic::wait is ~2x faster (no mutex overhead)
}

```

---

## Notes

- **Platform implementation:** `atomic::wait` uses the most efficient OS primitive — `futex` on Linux, `WaitOnAddress` on Windows. These are the same primitives that `condition_variable` uses internally.
- **Value comparison:** `wait(old)` compares using `memcmp` of the atomic's representation, not `operator==`. For most types, this is equivalent.
- **Spurious wakeups:** The C++ standard allows `wait()` to wake spuriously (without `notify`). The implementation handles this internally by re-checking the value.
- **`atomic_flag::wait()` (C++20):** Even simpler — `flag.wait(false)` blocks until the flag is set.
- **Use atomic::wait over sleep loops:** `while (!flag) sleep(1ms)` wastes up to 1ms of latency. `flag.wait(false)` wakes immediately when notified.
- Compile with `-std=c++20 -O2 -pthread`.
