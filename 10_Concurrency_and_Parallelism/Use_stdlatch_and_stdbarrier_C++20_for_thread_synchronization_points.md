# Use std::latch and std::barrier (C++20) for thread synchronization points

**Category:** Concurrency & Parallelism  
**Item:** #95  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/thread/latch>  

---

## Topic Overview

`std::latch` and `std::barrier` are thread coordination primitives for synchronizing groups of threads at a specific point.

### Comparison

```cpp

                  std::latch              std::barrier
─────────────────────────────────────────────────────────────
Uses:             Single-use              Reusable (multi-phase)
Counter:          Counts DOWN to 0        Resets after each phase
Threads:          Can count_down(n)       Each thread arrives once
                  multiple times          per phase
Wait:             arrive_and_wait()       arrive_and_wait()
Completion:       latch destroyed         Can run completion
                  after wait              function between phases
Use case:         "Wait for N inits"      "N threads, sync each step"

```

### API

| `std::latch` | Description |
| --- | --- |
| `latch(n)` | Construct with count n |
| `count_down(n=1)` | Decrement counter by n (don't wait) |
| `wait()` | Block until counter reaches 0 |
| `arrive_and_wait(n=1)` | `count_down(n)` + `wait()` in one call |
| `try_wait()` | Non-blocking check if counter == 0 |

| `std::barrier` | Description |
| --- | --- |
| `barrier(n, completion)` | Construct with n threads & optional completion fn |
| `arrive(n=1)` | Decrement counter (returns arrival token) |
| `wait(token)` | Block until phase completes |
| `arrive_and_wait()` | `arrive()` + `wait()` in one call |
| `arrive_and_drop()` | Arrive and permanently reduce participant count |

---

## Self-Assessment

### Q1: Use std::latch to wait for N threads to initialize before proceeding

**Answer:**

```cpp

#include <latch>
#include <thread>
#include <vector>
#include <iostream>
#include <chrono>
#include <string>

struct Service {
    std::string name;
    bool ready = false;

    void initialize() {
        // Simulate variable init time
        std::this_thread::sleep_for(
            std::chrono::milliseconds(50 + name.length() * 20));
        ready = true;
    }
};

int main() {
    constexpr int N = 4;
    std::vector<Service> services = {
        {"Database"}, {"Cache"}, {"Auth"}, {"Logger"}
    };

    std::latch init_latch(N); // count down from N

    std::vector<std::jthread> threads;
    for (int i = 0; i < N; ++i) {
        threads.emplace_back([&services, &init_latch, i] {
            services[i].initialize();
            std::cout << services[i].name << " initialized\n";
            init_latch.count_down(); // signal "I'm done"
            // Note: count_down does NOT block — thread continues
        });
    }

    // Main thread waits for ALL services to initialize
    init_latch.wait(); // blocks until counter == 0
    // ^^^ returns only after all N count_downs

    std::cout << "\nAll services ready:\n";
    for (const auto& s : services) {
        std::cout << "  " << s.name << ": " << (s.ready ? "OK" : "FAIL") << "\n";
    }

    // Output (order of init may vary):
    // Cache initialized
    // Auth initialized
    // Logger initialized
    // Database initialized
    //
    // All services ready:
    //   Database: OK
    //   Cache: OK
    //   Auth: OK
    //   Logger: OK

    // === arrive_and_wait: count_down + wait in one call ===
    std::cout << "\n--- Start gate pattern ---\n";
    std::latch start_gate(1); // main thread holds the gate

    std::vector<std::jthread> racers;
    for (int i = 0; i < 3; ++i) {
        racers.emplace_back([&start_gate, i] {
            start_gate.wait(); // all racers block here
            std::cout << "Racer " << i << " started!\n";
        });
    }

    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    std::cout << "GO!\n";
    start_gate.count_down(); // release all racers at once
}

```

### Q2: Implement a pipeline with std::barrier where threads synchronize at each stage

**Answer:**

```cpp

#include <barrier>
#include <thread>
#include <vector>
#include <iostream>
#include <functional>
#include <numeric>

// Parallel iterative computation:
// Each thread processes its chunk, then ALL threads synchronize
// before the next iteration begins.

int main() {
    constexpr int N_THREADS = 4;
    constexpr int N_PHASES = 3;
    constexpr int CHUNK = 4;

    std::vector<int> data(N_THREADS * CHUNK);
    std::iota(data.begin(), data.end(), 1); // 1, 2, 3, ..., 16

    int phase = 0;

    // Completion function runs ONCE between phases by the last arriving thread
    auto on_phase_complete = [&]() noexcept {
        ++phase;
        int sum = std::accumulate(data.begin(), data.end(), 0);
        std::cout << "--- Phase " << phase << " complete. Sum = " << sum << " ---\n";
    };

    std::barrier sync_point(N_THREADS, on_phase_complete);

    std::vector<std::jthread> workers;
    for (int t = 0; t < N_THREADS; ++t) {
        workers.emplace_back([&, t] {
            int start = t * CHUNK;
            int end = start + CHUNK;

            for (int p = 0; p < N_PHASES; ++p) {
                // Each thread processes its chunk
                for (int i = start; i < end; ++i) {
                    data[i] *= 2; // double each element
                }

                // Wait for ALL threads to finish this phase
                sync_point.arrive_and_wait();
                // ^^^ blocks until all N_THREADS call arrive_and_wait
                // Then on_phase_complete runs (by last thread)
                // Then all threads are released for next phase
            }
        });
    }

    // workers auto-join via jthread
    // After all phases:
    std::cout << "\nFinal: ";
    for (int x : data) std::cout << x << " ";
    std::cout << "\n";

    // Output:
    // --- Phase 1 complete. Sum = 272 ---
    // --- Phase 2 complete. Sum = 544 ---
    // --- Phase 3 complete. Sum = 1088 ---
    //
    // Final: 8 16 24 32 40 48 56 64 72 80 88 96 104 112 120 128
    // (each element multiplied by 2^3 = 8)
}

```

**Explanation:** The barrier syncs all threads at each phase boundary. The completion function runs atomically between phases — perfect for reduction, logging, or swapping buffers. After the completion function returns, all threads are released for the next phase.

### Q3: Explain why std::latch is a single-use synchronization primitive while std::barrier is reusable

**Answer:**

```cpp

LATCH vs BARRIER: Lifecycle
════════════════════════════

std::latch(3):
  count:  3 → 2 → 1 → 0  (done forever)
           ↓   ↓   ↓
          Thread arrivals
  
  After reaching 0: cannot be reset.
  Any thread calling wait() returns immediately (already at 0).
  Destroyed after use.

std::barrier(3):
  Phase 1: count 3 → 2 → 1 → 0 → [completion] → RESET to 3
  Phase 2: count 3 → 2 → 1 → 0 → [completion] → RESET to 3
  Phase 3: count 3 → 2 → 1 → 0 → [completion] → RESET to 3
  ... repeats indefinitely

  The barrier AUTOMATICALLY resets after each phase.
  The completion function runs between reset and release.

```

```cpp

#include <latch>
#include <barrier>
#include <thread>
#include <iostream>
#include <vector>

int main() {
    // === LATCH: Single-use — fire once, done ===
    {
        std::latch l(3);

        // Three threads count down
        std::jthread t1([&]{ l.count_down(); });
        std::jthread t2([&]{ l.count_down(); });
        std::jthread t3([&]{ l.count_down(); });

        l.wait(); // returns when counter hits 0

        // Counter is now 0. Calling wait() again? Returns immediately.
        l.wait(); // does NOT block (counter already 0)

        // But you CANNOT reset it to 3 and reuse it.
        // l.count_down(); // UB: counter would go below 0!

        std::cout << "Latch: triggered once, done.\n";
    }

    // === BARRIER: Multi-phase — automatically resets ===
    {
        int phase = 0;
        std::barrier b(3, [&]() noexcept {
            ++phase;
            std::cout << "Barrier: completed phase " << phase << "\n";
        });

        std::vector<std::jthread> threads;
        for (int i = 0; i < 3; ++i) {
            threads.emplace_back([&b] {
                b.arrive_and_wait(); // Phase 1
                b.arrive_and_wait(); // Phase 2 — barrier auto-reset!
                b.arrive_and_wait(); // Phase 3
            });
        }
        // threads auto-join

        std::cout << "Barrier: " << phase << " phases completed\n";
    }

    // === arrive_and_drop: dynamically reduce participants ===
    {
        std::barrier b(3, []() noexcept {
            std::cout << "Phase complete\n";
        });

        std::jthread t1([&] {
            b.arrive_and_wait(); // Phase 1
            b.arrive_and_drop(); // Leave: barrier now expects only 2
                                  // This thread must NOT call arrive again
        });

        std::jthread t2([&] {
            b.arrive_and_wait(); // Phase 1
            b.arrive_and_wait(); // Phase 2 (only 2 participants now)
        });

        std::jthread t3([&] {
            b.arrive_and_wait(); // Phase 1
            b.arrive_and_wait(); // Phase 2
        });
    }

    // Output:
    // Latch: triggered once, done.
    // Barrier: completed phase 1
    // Barrier: completed phase 2
    // Barrier: completed phase 3
    // Barrier: 3 phases completed
    // Phase complete
    // Phase complete
}

```

**Why the difference?**

- `latch` is designed for "wait until N things happen" — initialization gates, fan-in collection. Simple, lightweight, disposable.
- `barrier` is designed for iterative algorithms where threads must synchronize between steps — physics simulations, parallel sort passes, iterative solvers.

---

## Notes

- **Completion function requirements:** The barrier's completion function must be `noexcept` and must complete quickly (all threads are blocked waiting for it).
- **`count_down(n)` where n > 1:** A single thread can count down multiple times. Useful when one thread represents multiple logical events.
- **`arrive_and_drop()`:** Permanently reduces the barrier's expected count. The departing thread must not call `arrive_and_wait()` again.
- **Performance:** Both `latch` and `barrier` are typically built on `futex`/atomics — very efficient compared to manual `condition_variable` + counter implementations.
- **Latch vs counting semaphore:** A latch waits for the count to reach 0. A semaphore blocks when the count IS 0. Different semantics!
- Compile with `-std=c++20 -O2 -pthread`.
