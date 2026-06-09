# Understand GPU memory hierarchy: registers, shared memory, L1, L2, global

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#memory-hierarchy>  

---

## Topic Overview

### GPU Memory Hierarchy

The GPU memory hierarchy looks superficially similar to a CPU's, but the scale and implications are completely different. On a CPU you have maybe 8-32 cores, each with registers and a private cache. On a GPU you have thousands of threads, all competing for bandwidth, and the penalty for a cache miss is measured in hundreds of cycles. Understanding this hierarchy is the foundation of every performance optimization you'll ever make on a GPU.

Here's the full picture from fastest to slowest:

```cpp
Fastest --- Registers (per-thread, ~1 cycle)
    |       |
    |       Shared Memory / L1 Cache (per-SM, ~5-30 cycles)
    |       |
    |       L2 Cache (shared across all SMs, ~100-200 cycles)
    |       |
Slowest --- Global Memory (HBM/GDDR, ~300-800 cycles)
            |
            Host Memory (CPU RAM, ~10,000+ cycles via PCIe)
```

Every time your code misses a level and has to go further down this chain, you pay a latency penalty. The GPU's strategy for hiding that latency is to switch to another warp while the slow memory access completes - but that only works if you have enough warps active to keep the hardware busy.

### Memory Types in Detail

Here's the concrete data to go with the hierarchy. The bandwidth numbers are why GPU code obsesses over memory access patterns - even L2 bandwidth is measurably slower than shared memory:

| Memory | Scope | Size (typical) | Latency | Bandwidth |
| --- | --- | --- | --- | --- |
| **Registers** | Per-thread | ~255 per thread | 1 cycle | N/A |
| **Shared memory** | Per-SM (per-block) | 48-228 KB | ~5 cycles | ~19 TB/s |
| **L1 cache** | Per-SM | 128-256 KB | ~30 cycles | ~19 TB/s |
| **L2 cache** | All SMs | 4-96 MB | ~200 cycles | ~6 TB/s |
| **Global (HBM2e)** | All SMs | 16-80 GB | ~400 cycles | ~2 TB/s |
| **Host (PCIe 4.0)** | CPU+GPU | System RAM | ~10,000 cycles | ~32 GB/s |

Notice that global memory bandwidth (~2 TB/s) is only about a tenth of shared memory bandwidth. That 10x gap is why the tiling technique shown below produces such dramatic speedups.

### Registers — The Fastest Storage

Registers are per-thread, zero-cost to access, and your first choice for any value that a single thread works with repeatedly. Local variables live in registers automatically - you don't need to do anything special.

```cpp
__global__ void kernel(float* input, float* output, int n) {
    int idx = threadIdx.x + blockIdx.x * blockDim.x;  // Register
    float temp = input[idx];   // Register (after loading from global)
    temp = temp * temp + 1.0f; // Pure register operations -- fastest
    output[idx] = temp;        // Store back to global
}
```

The load from `input[idx]` and the store to `output[idx]` still touch global memory - there's no avoiding that. But the computation in the middle (`temp * temp + 1.0f`) is pure register work, which is essentially free from a latency perspective.

**Register pressure**: When a thread uses too many registers, the compiler **spills** to local memory (which, despite the name, is actually backed by global memory - very slow). Always check your register usage:

```bash
# Check register usage
nvcc --ptxas-options=-v kernel.cu
# "Used 32 registers" -- check this is reasonable
# If > ~64 registers per thread, consider reducing
```

Register spilling is silent and devastating. The compiler won't warn you - it just quietly starts using slow memory. This is one of the reasons `--ptxas-options=-v` should be part of your standard build during optimization.

### Shared Memory — Programmer-Controlled Cache

Shared memory is the most powerful tool in your GPU optimization toolkit. Think of it as a user-managed L1 cache: it's fast, it's visible to all threads in a block, and *you* decide what goes in it. The standard pattern is "tiling" - load a tile of data from slow global memory into fast shared memory, then do all the computation from shared memory.

The matrix multiplication example below demonstrates this clearly. Watch how each thread loads one element into the shared tile, then the entire tile is computed from shared memory instead of going back to global:

```cpp
__global__ void matrix_mul_shared(float* A, float* B, float* C,
                                   int N, int TILE) {
    __shared__ float tile_A[32][32];  // Shared memory -- 4 KB
    __shared__ float tile_B[32][32];

    int row = blockIdx.y * 32 + threadIdx.y;
    int col = blockIdx.x * 32 + threadIdx.x;
    float sum = 0.0f;

    for (int t = 0; t < N / 32; ++t) {
        // Cooperative load: each thread loads one element
        tile_A[threadIdx.y][threadIdx.x] = A[row * N + t * 32 + threadIdx.x];
        tile_B[threadIdx.y][threadIdx.x] = B[(t * 32 + threadIdx.y) * N + col];

        __syncthreads();  // Wait for all threads to finish loading

        // Compute from shared memory (fast!) instead of global (slow)
        for (int k = 0; k < 32; ++k) {
            sum += tile_A[threadIdx.y][k] * tile_B[k][threadIdx.x];
        }

        __syncthreads();  // Don't overwrite tile before others finish
    }

    C[row * N + col] = sum;
}
// Without tiling: ~30 GFLOPS
// With shared memory tiling: ~300 GFLOPS (10x improvement!)
```

The two `__syncthreads()` calls are critical. The first ensures all threads have finished loading before any thread starts computing. The second ensures all threads have finished computing before anyone starts overwriting the tile with the next batch. Forgetting either one produces subtle, hard-to-reproduce data races.

### Bank Conflicts in Shared Memory

Shared memory is divided into **banks** (typically 32 banks, 4 bytes each). If two threads in the same warp access different addresses that happen to map to the same bank, those accesses serialize - one thread waits for the other. This is called a bank conflict, and it's one of the most common performance problems in shared memory usage.

The reason this trips people up is that the pattern looks totally innocent in the source code. A stride-32 access looks like every thread is hitting a different index, but they all map to the same bank:

```cpp
// BAD -- bank conflict (all threads hit bank 0)
__shared__ float data[256];
float val = data[threadIdx.x * 32];  // Stride-32 = every access hits same bank

// GOOD -- no bank conflict (each thread hits a different bank)
float val = data[threadIdx.x];       // Stride-1 = sequential banks

// FIX for column access -- add padding
__shared__ float matrix[32][33];     // 33 instead of 32 -- shifts banks
float val = matrix[threadIdx.x][col]; // No bank conflict now
```

The padding trick (`[32][33]` instead of `[32][32]`) is a classic. By making the row width one element wider, each row starts at a different bank offset, and column accesses naturally hit different banks.

### Coalesced Global Memory Access

GPUs fetch global memory in **128-byte transactions**. If 32 threads in a warp all read consecutive addresses, the hardware satisfies them all in one transaction - that's "coalesced" access. If they read scattered addresses, you might need 32 separate transactions for 32 threads - 32x slower.

```cpp
// GOOD -- coalesced access (consecutive threads access consecutive addresses)
// Thread 0 reads x[0], Thread 1 reads x[1], ... Thread 31 reads x[31]
// -> One 128-byte transaction
float val = x[threadIdx.x + blockIdx.x * blockDim.x];

// BAD -- strided access (16x slower!)
// Thread 0 reads x[0], Thread 1 reads x[N], Thread 2 reads x[2*N]
// -> 32 separate transactions
float val = x[threadIdx.x * N];

// FIX: restructure data from AoS to SoA
// AoS (bad): struct { float x, y, z; } particles[N];  -> strided access
// SoA (good): float px[N], py[N], pz[N];               -> coalesced access
```

The Array of Structures vs Structure of Arrays distinction matters enormously here. With AoS, when 32 threads each try to read the `x` field of their particle, those `x` values are spaced `sizeof(Particle)` bytes apart - scattered across memory. With SoA, all the `x` values are packed together, so those 32 reads coalesce into one transaction.

### Memory Optimization Decision Tree

When you're deciding how to handle data in a kernel, this is the flow to work through:

```cpp
Is the data read-only?
├── Yes -> Use __constant__ memory (if < 64KB) or texture memory
└── No  -> Is it shared between threads in a block?
          ├── Yes -> Use __shared__ memory with tiling
          └── No  -> Use global memory with coalesced access patterns
```

### Profiling Memory Performance

After you've made changes to your memory access patterns, verify them with Nsight Compute. These three metrics together tell you the full story of your memory pipeline:

```bash
# NVIDIA Nsight Compute -- memory throughput analysis
ncu --metrics l1tex__t_sectors_pipe_lsu_mem_global_op_ld.sum,\
              l2__t_sectors_op_read.sum,\
              dram__sectors_read.sum \
    ./my_kernel

# Key metrics:
# - Global load efficiency: should be >50%
# - Shared memory bank conflicts: should be 0
# - L2 hit rate: higher is better
# - Achieved occupancy: should be >25%
```

If your global load efficiency is low, you have uncoalesced accesses. If bank conflicts are nonzero, you need padding. These two metrics together drive the vast majority of shared memory optimizations.

---

## Self-Assessment

### Q1: Why does Structure of Arrays (SoA) perform better than Array of Structures (AoS) on GPUs

GPUs access memory in 128-byte cache lines. With AoS, consecutive threads accessing the same field (e.g., `particles[i].x`) produce **strided** memory accesses - each thread's data is separated by `sizeof(Particle)` bytes, causing multiple memory transactions. With SoA, consecutive threads access consecutive memory addresses (`px[0], px[1], ..., px[31]`), fitting in a single 128-byte transaction. This **coalesced** access pattern can be 10-32x faster.

### Q2: What is a shared memory bank conflict and how do you avoid it

Shared memory is divided into 32 banks. When two threads in a warp access different addresses in the **same** bank simultaneously, the accesses serialize - one waits for the other. A stride of 32 elements is the worst case (all threads hit the same bank). Fix strategies: (1) use stride-1 access (natural indexing), (2) add padding to 2D arrays (`[32][33]` instead of `[32][32]`), (3) restructure the access pattern.

### Q3: When should you use shared memory vs rely on L1 cache

Use **shared memory** when: (1) data is reused multiple times by different threads in a block (tiling), (2) you need inter-thread communication within a block, (3) access patterns would cause L1 thrashing. Rely on **L1 cache** when: (1) data is accessed once or has simple patterns, (2) the working set fits in L1, (3) you want simpler code. Modern GPUs (Volta+) share the L1 and shared memory partition, so you can configure the split.

---

## Notes

- Always profile before optimizing - use `ncu` (Nsight Compute) for memory analysis; don't guess at what's slow.
- Register spilling is silent and devastating - check register count with `--ptxas-options=-v` and investigate if it's unexpectedly high.
- `cudaMemPrefetchAsync` can prime the L2 cache before kernel launch, which helps especially on the first kernel invocation when pages are cold.
- On AMD GPUs, the hierarchy is similar but uses LDS (Local Data Share) instead of shared memory - the same optimization principles apply.
- Unified Memory (`cudaMallocManaged`) simplifies programming by removing explicit copies, but may reduce peak performance due to page-fault overhead.
