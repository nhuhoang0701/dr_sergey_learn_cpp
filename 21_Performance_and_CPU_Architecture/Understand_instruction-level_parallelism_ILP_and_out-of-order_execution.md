# Understand instruction-level parallelism (ILP) and out-of-order execution

**Category:** Performance & CPU Architecture  
**Standard:** C++17 (architecture-aware coding)  
**Reference:** <https://en.wikipedia.org/wiki/Instruction-level_parallelism>  

---

## Topic Overview

### What Is ILP

**Instruction-Level Parallelism (ILP)** is the ability of a CPU to execute multiple instructions simultaneously by overlapping their execution stages. Modern CPUs are superscalar — they can dispatch 4-8+ instructions per cycle across multiple execution units.

### Out-of-Order Execution

Modern CPUs don't execute instructions in strict program order. The **out-of-order execution** engine:

1. **Fetches** instructions in order from the instruction stream.
2. **Decodes** them into micro-operations (µops).
3. **Renames** registers to eliminate false dependencies (WAR, WAW hazards).
4. **Dispatches** µops to execution ports as soon as operands are ready — regardless of original order.
5. **Retires** instructions in original program order (to maintain correctness).

```cpp

Program order:        Execution order (possible):

1. load  r1, [addr1]  → port 2 (cycle 1)
2. add   r2, r1, 5    → port 0 (cycle 5, waits for load)
3. load  r3, [addr2]  → port 3 (cycle 1, parallel with line 1!)
4. mul   r4, r3, 10   → port 1 (cycle 5, waits for load)

```

Instructions 1 and 3 are independent — the CPU executes them **in parallel**.

### Data Dependencies Limit ILP

```cpp

// Low ILP — each computation depends on the previous one (serial chain)
int serial(int a) {
    int x = a * 3;      // must wait for a
    int y = x + 7;      // must wait for x
    int z = y * 2;      // must wait for y
    return z + 1;       // must wait for z — total: 4 serial steps
}

// High ILP — independent computations can execute in parallel
int parallel(int a, int b, int c, int d) {
    int x = a * 3;   // independent
    int y = b + 7;   // independent — runs in parallel with x
    int z = c * 2;   // independent — runs in parallel
    int w = d + 1;   // independent — runs in parallel
    return x + y + z + w;  // depends on all, but computed in a tree
}

```

### Writing ILP-Friendly C++

**1. Break dependency chains:**

```cpp

// BAD: single accumulator — serial dependency chain
double sum_serial(const double* data, size_t n) {
    double sum = 0.0;
    for (size_t i = 0; i < n; ++i)
        sum += data[i];  // each add depends on previous sum
    return sum;
}

// GOOD: multiple accumulators — independent chains
double sum_parallel(const double* data, size_t n) {
    double sum0 = 0.0, sum1 = 0.0, sum2 = 0.0, sum3 = 0.0;
    size_t i = 0;
    for (; i + 3 < n; i += 4) {
        sum0 += data[i];      // independent chain
        sum1 += data[i + 1];  // independent chain
        sum2 += data[i + 2];  // independent chain
        sum3 += data[i + 3];  // independent chain
    }
    for (; i < n; ++i) sum0 += data[i];
    return sum0 + sum1 + sum2 + sum3;
}

```

**2. Avoid long latency chains:**

```cpp

// Division has high latency (~20-40 cycles) — avoid chaining divisions
double bad(double a, double b, double c) {
    return a / b / c;  // serial: div → div
}

double better(double a, double b, double c) {
    return a / (b * c);  // multiply is faster, then one division
}

```

### The Reorder Buffer (ROB) and Its Limits

The CPU's **Reorder Buffer** holds in-flight instructions. On modern x86:

- Intel Alder Lake: ~512 entries
- AMD Zen 4: ~320 entries

If the ROB fills up (due to a long-latency stall like a cache miss), the CPU cannot issue new instructions — even if they're independent. This is why **reducing cache misses** also improves ILP.

---

## Self-Assessment

### Q1: Why do multiple accumulators improve performance

With a single accumulator, each `sum += data[i]` depends on the previous value of `sum` — creating a serial dependency chain with latency = N × add_latency.

With 4 accumulators, each chain is independent. The CPU's out-of-order engine dispatches additions from all 4 chains in parallel, achieving up to 4× throughput. The final reduction `sum0 + sum1 + sum2 + sum3` is negligible.

### Q2: What is register renaming and why does it help

```assembly

; Without renaming — false dependency (WAW hazard):
mov eax, [addr1]    ; write eax
add eax, 5          ; read eax, write eax
mov eax, [addr2]    ; write eax — must wait even though values are unrelated!

; With renaming — CPU uses different physical registers:
mov r100, [addr1]   ; physical r100
add r101, r100, 5   ; physical r101
mov r102, [addr2]   ; physical r102 — NO dependency on r100/r101!

```

Register renaming eliminates false dependencies (Write-After-Read, Write-After-Write), allowing independent instruction sequences to execute in parallel even when they use the same architectural registers.

### Q3: Show a code pattern that kills ILP and how to fix it

```cpp

// ILP killer: linked-list traversal — each step depends on the previous pointer
Node* ptr = head;
int count = 0;
while (ptr) {
    count += ptr->value;
    ptr = ptr->next;    // pointer-chasing: next address unknown until load completes
}

// Fix: use contiguous data (std::vector) for predictable access patterns
std::vector<int> values = /* ... */;
int count = 0;
for (int v : values) {
    count += v;  // hardware prefetcher predicts sequential accesses
}

```

---

## Notes

- Modern CPUs can typically dispatch 4-8 µops per cycle — but only if there are enough independent instructions.
- Compilers help with ILP (instruction scheduling), but algorithmic data dependencies are fundamental.
- Profile with `perf stat` and check IPC (instructions per cycle) — IPC < 1.0 usually means stalls.
- Multiple accumulators, loop unrolling, and SoA (struct-of-arrays) layouts all improve ILP.
