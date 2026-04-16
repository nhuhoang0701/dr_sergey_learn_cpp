# Understand micro-architectural bottlenecks: port pressure, ROB size, store buffers

**Category:** Performance & CPU Architecture  
**Standard:** C++17 (architecture-aware coding)  
**Reference:** <https://uops.info/> <https://agner.org/optimize/>  

---

## Topic Overview

### Beyond Big-O: Micro-Architecture Matters

Two algorithms with the same big-O complexity can differ by 10× in performance due to **micro-architectural** effects. Understanding the CPU pipeline is essential for low-level optimization.

### Execution Ports and Port Pressure

Modern CPUs have multiple **execution ports**, each capable of handling specific types of µops:

```cpp

Intel Alder Lake P-core (Golden Cove):
Port 0: ALU, Vec ALU, Branch
Port 1: ALU, Vec ALU, Integer Mul
Port 2: Load (AGU)
Port 3: Load (AGU)
Port 4: Store Data
Port 5: ALU, Vec ALU, Vec Shuffle
Port 6: ALU, Branch
Port 7: Store AGU
Port 8: Store AGU
Port 9: Store Data

```

**Port pressure** occurs when too many µops compete for the same port, creating a bottleneck even though other ports are idle.

```cpp

// Example: high port 0/1 pressure (ALU-heavy, no memory ops)
for (int i = 0; i < n; ++i) {
    result[i] = a[i] * b[i] + c[i] * d[i];
    // All multiplications compete for port 1 (integer multiply)
}

// Better: interleave with memory operations to balance port utilization
for (int i = 0; i < n; ++i) {
    auto ai = a[i];  // load — port 2/3
    auto bi = b[i];  // load — port 2/3
    auto ab = ai * bi;  // multiply — port 1
    auto ci = c[i];  // load — port 2/3 (overlaps with multiply)
    auto di = d[i];  // load
    result[i] = ab + ci * di;  // add — port 0/5, multiply — port 1
}

```

### Reorder Buffer (ROB) Size

The ROB tracks all in-flight µops from decode to retirement. If it fills up, the front end stalls.

| CPU | ROB Size |
| --- | --- |
| Intel Alder Lake (P-core) | 512 entries |
| Intel Raptor Lake (P-core) | 512 entries |
| AMD Zen 4 | 320 entries |
| AMD Zen 5 | 448 entries |
| Apple M3 (Avalanche) | ~630 entries |

**When the ROB fills:**

- A long-latency operation (cache miss: ~200 cycles for DRAM) blocks retirement.
- The ROB fills with younger µops waiting behind the stalled instruction.
- The front end stops fetching new instructions.
- The entire pipeline stalls.

```cpp

// ROB-unfriendly: unpredictable memory access → cache misses → ROB fills
for (int i = 0; i < n; ++i) {
    sum += data[indices[i]];  // random access → L3/DRAM miss → ROB stall
}

// ROB-friendly: prefetch to hide latency, keeping ROB from filling
for (int i = 0; i < n; ++i) {
    __builtin_prefetch(&data[indices[i + 16]]);  // prefetch future access
    sum += data[indices[i]];  // hopefully in cache by now
}

```

### Store Buffer

The **store buffer** holds stores before they commit to the cache hierarchy. It:

- Allows stores to retire before writing to cache (write combining).
- Supports store-to-load forwarding (a load can read from a pending store).
- Has limited entries (~56-72 on modern x86).

**Store buffer stalls:**

```cpp

// Many stores can fill the store buffer
void fill_array(int* arr, int n) {
    for (int i = 0; i < n; ++i)
        arr[i] = i;  // each iteration = 1 store
    // If stores are to different cache lines, they cannot be combined
    // Store buffer may fill, causing a stall
}

// Non-temporal stores bypass the store buffer and cache
#include <immintrin.h>
void fill_array_nt(int* arr, int n) {
    for (int i = 0; i < n; i += 4) {
        __m128i val = _mm_set_epi32(i+3, i+2, i+1, i);
        _mm_stream_si128((__m128i*)&arr[i], val);  // bypasses cache
    }
}

```

### Detecting Bottlenecks with `perf`

```bash

# Top-down micro-architecture analysis
perf stat -e \
  topdown-retiring,topdown-bad-spec,topdown-fe-bound,topdown-be-bound \
  ./my_app

# Typical output:
# 35.2% topdown-retiring      ← useful work
# 5.1%  topdown-bad-spec      ← branch mispredictions
# 12.3% topdown-fe-bound      ← front-end stalls (instruction fetch/decode)
# 47.4% topdown-be-bound      ← back-end stalls (memory, execution units) ← bottleneck

```

---

## Self-Assessment

### Q1: What causes a front-end stall vs. a back-end stall

- **Front-end stall**: The instruction fetch/decode pipeline can't keep up. Causes: I-cache miss, complex instructions, branch mispredictions, instruction alignment issues.
- **Back-end stall**: The execution engine can't make progress. Causes: D-cache misses (memory bound), port pressure (compute bound), ROB full, store buffer full.

### Q2: How does store-to-load forwarding work

When a load reads from an address that has a pending store in the store buffer, the CPU **forwards** the data directly from the store buffer to the load — without waiting for the store to commit to cache. This works when the load completely overlaps the store's data. Partial overlaps cause a forwarding failure and a pipeline flush.

### Q3: How do you identify port pressure with perf

```bash

# On Intel, use specific port utilization counters
perf stat -e uops_dispatched_port.port_0,\
uops_dispatched_port.port_1,\
uops_dispatched_port.port_2,\
uops_dispatched_port.port_3,\
uops_dispatched_port.port_5,\
uops_dispatched_port.port_6 \
./my_app

```

If one port has significantly more µops than others, that port is the bottleneck. Restructure the code to use operations that map to less-busy ports.

---

## Notes

- Use <https://uops.info/> to check which ports specific instructions use.
- Agner Fog's instruction tables are the definitive reference for latency/throughput per instruction.
- Intel VTune's "Microarchitecture Exploration" analysis automates bottleneck detection.
- Compiler flags like `-march=native` allow the compiler to use all available ports and instructions.
