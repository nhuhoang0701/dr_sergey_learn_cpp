# Abstract platform-specific atomic and memory model differences

**Category:** Cross-Platform Development  
**Standard:** C++17/20  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/memory_order>  

---

## Topic Overview

### The C++ Memory Model vs Hardware Reality

The C++ memory model is an abstraction. Real CPUs have different hardware memory models:

| Architecture | Model | Key characteristic |
| --- | --- | --- |
| **x86/x86-64** | TSO (Total Store Ordering) | Strong — stores are ordered, loads are ordered, only store-load reordering possible |
| **ARM (v7/v8)** | Weak | Loads and stores can be reordered freely |
| **RISC-V** | Weak (RVWMO) | Similar to ARM — requires explicit fences |
| **POWER/PowerPC** | Very Weak | Even dependent loads can be reordered |

### Why This Matters for Cross-Platform Code

Code that works on x86 may break on ARM because x86 provides stronger ordering guarantees than the C++ standard requires:

```cpp

// This WORKS on x86 but is BROKEN on ARM!
std::atomic<bool> ready{false};
int data = 0;

// Thread 1
void producer() {
    data = 42;
    ready.store(true, std::memory_order_relaxed);  // No fence!
}

// Thread 2
void consumer() {
    while (!ready.load(std::memory_order_relaxed)) {}
    int x = data;  // On ARM: might read 0 (stale data)!
                    // On x86: always reads 42 (TSO guarantees it)
}

```

The fix: use proper memory ordering that works on ALL architectures:

```cpp

// CORRECT — portable across all architectures
void producer() {
    data = 42;
    ready.store(true, std::memory_order_release);  // Release fence
}

void consumer() {
    while (!ready.load(std::memory_order_acquire)) {}  // Acquire fence
    int x = data;  // Guaranteed to see 42 on ALL platforms
}

```

### Memory Order Mapping to Hardware

| C++ memory order | x86 | ARM v8 | RISC-V |
| --- | --- | --- | --- |
| `relaxed` | MOV (plain) | LDR/STR (plain) | LW/SW (plain) |
| `acquire` (load) | MOV (free — TSO gives this) | LDAR | LW; FENCE R,RW |
| `release` (store) | MOV (free — TSO gives this) | STLR | FENCE RW,W; SW |
| `acq_rel` | MOV (free) | LDAR/STLR | Full FENCE |
| `seq_cst` (load) | MOV (free) | LDAR | FENCE RW,RW; LW |
| `seq_cst` (store) | MOV + MFENCE (or XCHG) | STLR | SW; FENCE RW,RW |

Key insight: on x86, most orderings are **free** because the hardware already provides TSO. On ARM, every ordering beyond `relaxed` requires a hardware fence instruction.

### Abstraction Pattern: Lock-Free Queue

```cpp

#include <atomic>
#include <array>
#include <optional>

template<typename T, size_t N>
class SPSCQueue {  // Single-producer, single-consumer
    std::array<T, N> buffer_;
    // Use alignas to prevent false sharing
    alignas(64) std::atomic<size_t> head_{0};
    alignas(64) std::atomic<size_t> tail_{0};

public:
    bool push(const T& value) {
        size_t tail = tail_.load(std::memory_order_relaxed);  // Only producer writes tail
        size_t next = (tail + 1) % N;

        if (next == head_.load(std::memory_order_acquire)) {  // Acquire: sync with consumer
            return false;  // Full
        }

        buffer_[tail] = value;
        tail_.store(next, std::memory_order_release);  // Release: publish the data
        return true;
    }

    std::optional<T> pop() {
        size_t head = head_.load(std::memory_order_relaxed);  // Only consumer writes head

        if (head == tail_.load(std::memory_order_acquire)) {  // Acquire: sync with producer
            return std::nullopt;  // Empty
        }

        T value = buffer_[head];
        head_.store((head + 1) % N, std::memory_order_release);  // Release: free the slot
        return value;
    }
};
// This works identically on x86, ARM, and RISC-V

```

### Testing on Weak Memory Architectures

```bash

# Cross-compile for ARM and run under QEMU
aarch64-linux-gnu-g++ -std=c++20 -O2 -pthread test.cpp -o test_arm
qemu-aarch64 -L /usr/aarch64-linux-gnu test_arm

# Use ThreadSanitizer (catches ordering bugs even on x86)
g++ -std=c++20 -O2 -fsanitize=thread test.cpp -o test_tsan
./test_tsan

```

ThreadSanitizer is particularly valuable because it detects data races that would be hidden by x86's strong ordering but would fail on ARM.

### Common Cross-Platform Atomic Pitfalls

```cpp

// Pitfall 1: Assuming atomic operations are always lock-free
std::atomic<LargeStruct> a;  // May use a mutex internally!
static_assert(std::atomic<LargeStruct>::is_always_lock_free,
              "Need lock-free atomics for performance");

// Pitfall 2: Different alignment requirements
struct Data {
    std::atomic<int64_t> value;  // Must be 8-byte aligned
    // On some 32-bit ARM, misaligned atomic<int64_t> is not lock-free
};
static_assert(alignof(std::atomic<int64_t>) >= 8);

// Pitfall 3: Using volatile instead of atomic
volatile int flag = 0;  // NOT atomic! No ordering guarantees!
std::atomic<int> proper_flag{0};  // Use this instead

// Pitfall 4: Relying on memory_order_relaxed for synchronization
std::atomic<int> sync_flag{0};
sync_flag.store(1, std::memory_order_relaxed);  // No synchronization!
// Use release/acquire for producer-consumer patterns

```

---

## Self-Assessment

### Q1: Why does relaxed ordering work "by accident" on x86 but fail on ARM

x86 uses TSO (Total Store Ordering), which guarantees:

- Loads are not reordered with other loads
- Stores are not reordered with other stores
- Loads are not reordered with earlier stores

The only reordering x86 allows is a later load before an earlier store. This means `memory_order_acquire` and `memory_order_release` are effectively free on x86 — the hardware already provides them. Code using `relaxed` ordering often accidentally works because TSO provides stronger guarantees.

ARM uses a weak memory model where loads and stores can be reordered in any direction. `relaxed` really means "no ordering" — the CPU will reorder aggressively for performance. You must use explicit acquire/release to get the same behavior x86 gives you for free.

### Q2: How do you verify your lock-free code is correct on weak architectures

1. **ThreadSanitizer** (`-fsanitize=thread`) — detects data races, works on x86
2. **ARM cross-compilation + QEMU** — actually runs on a weak memory model
3. **CDSChecker** or **GenMC** — model checkers that explore all possible memory orderings
4. **Stress testing** with many threads — increases the chance of exposing reordering bugs

### Q3: What is the cost of `seq_cst` vs `relaxed` on each platform

On x86: `seq_cst` stores require an `MFENCE` instruction (~20-40 cycles). `relaxed` and `acquire`/`release` are free.

On ARM: `seq_cst` uses `LDAR`/`STLR` + full barrier. `acquire`/`release` uses `LDAR`/`STLR` (~5-10 cycles). `relaxed` is free. The relative cost is smaller on ARM because even weaker orderings need instructions.

Rule of thumb: use `acquire`/`release` for producer-consumer patterns and `seq_cst` only when you need total ordering across multiple atomics.

---

## Notes

- Always develop with ThreadSanitizer enabled — it catches bugs invisible on x86
- Default to `memory_order_seq_cst` (the default) unless you have measured a performance need for weaker ordering
- `std::atomic_ref` (C++20) lets you do atomic operations on non-atomic variables — same ordering rules apply
- The `std::hardware_destructive_interference_size` constant helps avoid false sharing portably
- When in doubt, use `seq_cst` — it is correct on all platforms and usually fast enough
