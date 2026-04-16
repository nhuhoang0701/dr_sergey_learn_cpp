# Know how volatile differs from atomic and when each is appropriate

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/cv>  

---

## Topic Overview

### volatile vs. atomic — Fundamentally Different

These two features serve completely different purposes despite both preventing certain optimizations:

| Feature | `volatile` | `std::atomic<T>` |
| --- | --- | --- |
| **Purpose** | Prevent compiler from optimizing away memory accesses | Thread-safe access to shared data |
| **Atomicity** | ❌ No guarantee | ✅ Guaranteed (lock-free or with internal lock) |
| **Memory ordering** | ❌ No ordering between threads | ✅ Sequential consistency by default |
| **Prevents reordering** | Only compiler reordering of volatile accesses | Both compiler and CPU reordering |
| **Thread safety** | ❌ Not safe for inter-thread communication | ✅ Designed for multi-threaded code |
| **Use case** | Hardware registers, signal handlers | Shared flags, counters, lock-free data structures |

### What `volatile` Actually Does

`volatile` tells the compiler: **"The value of this variable may change at any time for reasons outside your knowledge. Do not cache it in a register; always read from / write to memory."**

```cpp

volatile int* hardware_register = reinterpret_cast<volatile int*>(0x40021000);

// Without volatile: compiler may optimize to a single read
// With volatile: compiler reads from memory address EVERY time
int status = *hardware_register;     // Forced memory read #1
int status2 = *hardware_register;    // Forced memory read #2 (may differ!)

```

**What `volatile` does NOT do:**

- Does not prevent CPU reordering of instructions
- Does not make reads/writes atomic (a 64-bit write may be two 32-bit writes)
- Does not provide happens-before relationships between threads
- Does not prevent data races — using `volatile` for threading is UB

### What `std::atomic` Does

`std::atomic` provides:

1. **Atomicity** — operations are indivisible
2. **Memory ordering** — establishes happens-before relationships
3. **Visibility** — writes in one thread are visible to reads in another

```cpp

#include <atomic>
std::atomic<int> counter{0};

// Thread 1:
counter.fetch_add(1);  // Atomic increment — indivisible, visible to all threads

// Thread 2:
int val = counter.load();  // Guaranteed to see Thread 1's write (or later)

```

### C++20: Deprecated `volatile` Compound Operations

C++20 deprecates many `volatile` operations that never made sense:

```cpp

volatile int v = 0;
v++;        // DEPRECATED in C++20 (compound assignment on volatile)
v += 5;     // DEPRECATED in C++20
int x = v;  // OK — simple read
v = 42;     // OK — simple write

```

---

## Self-Assessment

### Q1: Explain that `volatile` prevents optimization of reads/writes but does not provide atomicity or ordering

**`volatile` prevents compiler optimizations on memory access:**

```cpp

#include <iostream>

volatile int sensor_value = 0;

void read_sensor() {
    // Without volatile: compiler could optimize this to a single read or even
    // replace with a constant if it proves sensor_value isn't written in this function
    int a = sensor_value;   // Forced read from memory
    int b = sensor_value;   // Another forced read from memory (may differ!)
    int c = sensor_value;   // Yet another forced read

    // With a non-volatile int, the compiler could do:
    // int a = sensor_value;  // One read
    // int b = a;             // Reuse cached value
    // int c = a;             // Reuse cached value
    std::cout << a << " " << b << " " << c << "\n";
}

// But volatile provides NO atomicity:
volatile long long big_val = 0;
// On a 32-bit system, reading big_val may require two 32-bit loads.
// Another thread could modify big_val between the two loads → torn read!

// And volatile provides NO ordering:
volatile int x = 0, y = 0;
// Thread 1:          Thread 2:
// x = 1;             if (y == 1)
// y = 1;                 assert(x == 1); // MAY FAIL! CPU can reorder

```

**Key takeaways:**

- `volatile` only affects the **compiler** — it forces memory reads/writes but doesn't issue CPU fence instructions.
- On modern CPUs, reads and writes can be **reordered by hardware** — `volatile` doesn't prevent this.
- A 64-bit `volatile` variable on a 32-bit platform can be **torn** (read/written in two halves).
- `volatile` alone is **never sufficient** for thread-safe communication.

### Q2: Show a case where `volatile` is appropriate: memory-mapped hardware registers

```cpp

#include <cstdint>

// Memory-mapped I/O for an embedded UART peripheral
struct UART_Registers {
    volatile uint32_t DR;     // Data register — changes when data arrives
    volatile uint32_t SR;     // Status register — hardware updates these bits
    volatile uint32_t CR;     // Control register — writing changes hardware behavior
    volatile uint32_t BRR;    // Baud rate register
};

// Map to hardware address (platform-specific)
auto* const uart1 = reinterpret_cast<UART_Registers*>(0x40011000);

void uart_send_byte(uint8_t byte) {
    // Wait for transmit buffer empty — SR changes by HARDWARE
    // Without volatile, compiler would read SR once and loop forever
    while (!(uart1->SR & (1 << 7))) {
        // busy-wait: SR is volatile, so it's re-read every iteration
    }
    uart1->DR = byte;  // Write to data register triggers hardware transmission
}

uint8_t uart_receive_byte() {
    // Wait for receive buffer not empty
    while (!(uart1->SR & (1 << 5))) {
        // busy-wait: must re-read SR from hardware every time
    }
    return static_cast<uint8_t>(uart1->DR);  // Read clears the RXNE flag in hardware
}

// Another valid use: signal handlers (limited)
#include <csignal>

volatile std::sig_atomic_t signal_received = 0;

void signal_handler(int sig) {
    signal_received = 1;  // volatile ensures the write isn't optimized away
}

int main() {
    std::signal(SIGINT, signal_handler);
    while (!signal_received) {
        // volatile ensures signal_received is re-read each iteration
    }
    return 0;
}

```

**Why `volatile` is correct here:**

- Hardware registers change **independently** of program execution — the compiler must not cache them.
- There's only one thread accessing the registers (bare-metal code), so atomicity isn't a concern.
- For signal handlers, `volatile sig_atomic_t` is the standard-mandated type.

### Q3: Show a case where `volatile` is wrong but `std::atomic` is correct

```cpp

#include <atomic>
#include <thread>
#include <iostream>
#include <vector>

// ===== WRONG: using volatile for inter-thread communication =====
namespace wrong {
    volatile bool stop_flag = false;  // volatile: NO atomicity, NO ordering!
    volatile int shared_data = 0;

    void producer() {
        shared_data = 42;     // Write data
        stop_flag = true;     // Signal "data is ready"
        // BUG: CPU may reorder these writes! Consumer may see stop_flag=true
        // but shared_data still 0
    }

    void consumer() {
        while (!stop_flag) {} // Spin-wait
        // BUG: even if we see stop_flag==true, shared_data may not be 42 yet
        std::cout << "wrong: " << shared_data << "\n";  // May print 0!
    }
}

// ===== CORRECT: using std::atomic =====
namespace correct {
    std::atomic<bool> stop_flag{false};
    int shared_data = 0;  // Protected by the atomic flag (acquire-release)

    void producer() {
        shared_data = 42;                          // Write data
        stop_flag.store(true, std::memory_order_release);  // Release: all prior writes visible
    }

    void consumer() {
        while (!stop_flag.load(std::memory_order_acquire)) {} // Acquire: sees all writes before release
        std::cout << "correct: " << shared_data << "\n";  // GUARANTEED to print 42
    }
}

// ===== WRONG: using volatile for a shared counter =====
namespace counter_wrong {
    volatile int count = 0;

    void increment() {
        for (int i = 0; i < 100000; ++i)
            count++;  // NOT atomic! Read-modify-write is 3 steps: load, add, store
                      // Two threads can load the same value → lost updates
    }
}

// ===== CORRECT: using atomic for a shared counter =====
namespace counter_correct {
    std::atomic<int> count{0};

    void increment() {
        for (int i = 0; i < 100000; ++i)
            count.fetch_add(1, std::memory_order_relaxed);  // Atomic RMW operation
    }
}

int main() {
    // Test the correct flag pattern:
    {
        std::thread t1(correct::producer);
        std::thread t2(correct::consumer);
        t1.join(); t2.join();
        // Always prints: correct: 42
    }

    // Test the correct counter:
    {
        std::vector<std::thread> threads;
        for (int i = 0; i < 4; ++i)
            threads.emplace_back(counter_correct::increment);
        for (auto& t : threads) t.join();
        std::cout << "Atomic counter: " << counter_correct::count << "\n";
        // Always prints: 400000
    }

    return 0;
}

```

**Why `volatile` fails for threading:**

1. **No atomicity:** `count++` on a `volatile int` compiles to load → add → store. Two threads can execute these steps interleaved, causing lost updates.
2. **No memory ordering:** The CPU can reorder `shared_data = 42` after `stop_flag = true`. The consumer thread may see the flag set but read stale data.
3. **No guarantee of visibility:** On some architectures, writes to non-atomic variables may sit in a store buffer and never become visible to other cores.

**Why `std::atomic` works:**

1. `fetch_add` is an atomic read-modify-write — indivisible.
2. `store(val, release)` + `load(acquire)` creates a happens-before relationship.
3. All writes before a release store are guaranteed visible after an acquire load.

---

## Additional Examples

### When You Need Both volatile AND atomic

In rare embedded scenarios, you need both (e.g., memory-mapped atomic hardware register):

```cpp

// C++11 doesn't support volatile atomic directly.
// C++20 deprecated volatile operations on atomics.
// For hardware registers that need both:
#include <atomic>

// Use volatile pointer to atomic:
volatile std::atomic<uint32_t>* hw_semaphore =
    reinterpret_cast<volatile std::atomic<uint32_t>*>(0x40002000);

// Or better: use atomic_ref (C++20) with a volatile access:

```

### Summary Decision Table

| Scenario | Use `volatile`? | Use `std::atomic`? |
| --- | --- | --- |
| Memory-mapped hardware register | ✅ Yes | ❌ No |
| Signal handler flag | ✅ `volatile sig_atomic_t` | ❌ No |
| Shared flag between threads | ❌ No | ✅ Yes |
| Shared counter between threads | ❌ No | ✅ Yes |
| Lock-free queue | ❌ No | ✅ Yes |
| DMA buffer | ✅ Yes | Depends |

---

## Notes

- **`volatile` ≠ thread-safe.** This is the #1 misconception in C++.
- Java/C# `volatile` provides atomicity + ordering. C++ `volatile` does NOT.
- C++20 deprecated compound operations on `volatile` (e.g., `volatile_var++`).
- For threading: always use `std::atomic`, `std::mutex`, or other synchronization primitives.
- For hardware: use `volatile` (and nothing else — `std::atomic` may insert unnecessary fences).
