# Profile GPU kernels with NSight and ROCProfiler

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://developer.nvidia.com/nsight-compute>  

---

## Topic Overview

### Why GPU Profiling Is Different

CPU profiling tools (perf, VTune) cannot see inside GPU kernels. GPU-specific profilers are needed because:

- GPU execution is **asynchronous** — kernel launch ≠ kernel execution
- Thousands of threads run simultaneously — bottlenecks differ from CPU
- Memory hierarchy is different — shared memory, coalescing, bank conflicts
- Occupancy, warp divergence, and instruction mix matter

### NVIDIA Profiling Tool Stack

| Tool | Purpose | Level |
| --- | --- | --- |
| **Nsight Systems** (`nsys`) | System-wide timeline: CPU+GPU, API calls, kernel launches | High-level |
| **Nsight Compute** (`ncu`) | Per-kernel deep analysis: memory, compute, occupancy | Low-level |
| **CUPTI** | Programmatic profiling API | Library |
| **nvprof** (legacy) | Deprecated — use nsys/ncu instead | — |

### Nsight Systems — Finding the Right Kernel to Optimize

```bash

# Capture a system-wide trace
nsys profile --stats=true ./my_app

# Output: timeline showing CPU and GPU activity
# Look for:
# - Large gaps between kernel launches (CPU overhead)
# - Long-running kernels (optimization targets)
# - Excessive cudaMemcpy (data transfer bottleneck)

# Export to file for GUI analysis
nsys profile -o my_trace ./my_app
# Open my_trace.nsys-rep in Nsight Systems GUI

```

The timeline view shows:

```cpp

CPU Thread 0: [cudaMalloc][cudaMemcpy][<<<kernel_A>>>][<<<kernel_B>>>][cudaMemcpy]
GPU Stream 0:              [memcpy H2D] [kernel_A ████████][kernel_B ██][memcpy D2H]
                            idle ↑           ↑ target for optimization

```

### Nsight Compute — Deep Kernel Analysis

```bash

# Profile a specific kernel
ncu --kernel-name vector_add --launch-skip 0 --launch-count 1 ./my_app

# Full metrics collection (slower but comprehensive)
ncu --set full --kernel-name matrix_mul ./my_app

# Specific metrics
ncu --metrics \
    sm__throughput.avg.pct_of_peak_sustained_elapsed,\
    gpu__compute_memory_throughput.avg.pct_of_peak_sustained_elapsed,\
    l1tex__t_sectors_pipe_lsu_mem_global_op_ld.sum,\
    sm__warps_active.avg.pct_of_peak_sustained_elapsed \
    ./my_app

# Compare two implementations
ncu --set full -o baseline ./my_app_v1
ncu --set full -o optimized ./my_app_v2
ncu --diff baseline.ncu-rep optimized.ncu-rep

```

### Key Nsight Compute Metrics Explained

```cpp

┌─────────────────────────────────────────────┐
│ Kernel: matrix_mul  Grid: [128,128,1]       │
│ Block: [32,32,1]    Duration: 1.2 ms        │
├─────────────────────────────────────────────┤
│ Compute (SM) Throughput:        45.2%       │ ← Not compute-bound
│ Memory Throughput:              87.3%       │ ← Memory-bound!
│ Achieved Occupancy:             75.0%       │ ← Good (>50%)
│                                             │
│ Global Load Efficiency:         25.0%       │ ← BAD! Uncoalesced
│ Shared Memory Bank Conflicts:   1.2M/s     │ ← Needs padding fix
│ Warp Execution Efficiency:      62.5%       │ ← Branch divergence
│ Register Spills (stores):       8/thread    │ ← Register pressure
└─────────────────────────────────────────────┘

```

Interpretation:

- **Memory-bound** (87.3% memory throughput) → optimize memory access patterns
- **Low global load efficiency** (25%) → data is not coalesced, use SoA
- **Bank conflicts** → add padding to shared memory arrays
- **Low warp efficiency** (62.5%) → reduce branch divergence

### AMD ROCProfiler for AMD GPUs

```bash

# Install ROCm profiling tools
sudo apt install rocprofiler-dev

# Basic kernel profiling
rocprof --stats ./my_hip_app
# Output: results.stats.csv with kernel times

# Collect hardware counters
cat > input.txt << EOF
pmc : SQ_WAVES SQ_INSTS_VALU SQ_INSTS_SMEM
kernel : matrix_mul
EOF
rocprof -i input.txt ./my_hip_app

# Key AMD GPU metrics:
# SQ_WAVES          — total wavefronts launched
# SQ_INSTS_VALU     — vector ALU instructions
# SQ_INSTS_SMEM     — scalar memory instructions
# TA_TOTAL_WAVEFRONTS_SENT — texture/memory wavefronts
# TCP_TCC_READ_REQ  — L2 cache read requests

```

### AMD Omniperf — High-Level Analysis

```bash

# Profile and analyze
omniperf profile -n my_run -- ./my_hip_app
omniperf analyze -p workloads/my_run/

# Generates a report with:
# - Roofline analysis
# - Memory chart (L1, L2, HBM utilization)
# - Compute utilization
# - Recommendations for optimization

```

### Embedding Profiling in C++ Code

```cpp

// NVIDIA: NVTX annotations for Nsight timeline
#include <nvtx3/nvToolsExt.h>

void process_frame() {
    nvtxRangePush("Data Preparation");
    prepare_data();
    nvtxRangePop();

    nvtxRangePush("GPU Computation");
    launch_kernel<<<grid, block>>>(data);
    nvtxRangePop();

    nvtxRangePush("Result Download");
    cudaMemcpy(host_result, dev_result, size, cudaMemcpyDeviceToHost);
    nvtxRangePop();
}

// CUDA Events for timing
cudaEvent_t start, stop;
cudaEventCreate(&start);
cudaEventCreate(&stop);

cudaEventRecord(start);
my_kernel<<<grid, block>>>(args);
cudaEventRecord(stop);
cudaEventSynchronize(stop);

float ms = 0;
cudaEventElapsedTime(&ms, start, stop);
printf("Kernel time: %.3f ms\n", ms);

cudaEventDestroy(start);
cudaEventDestroy(stop);

```

### The Roofline Model

The roofline model helps determine if a kernel is compute-bound or memory-bound:

```cpp

          Peak Compute ─────────────────────────────────
         /                                              
        /     ← Compute-bound region                    
       /                                                
Peak ─/──────── Your kernel (*)                         
     /                                                  
    /  ← Memory-bound region                            
   /                                                    
  /                                                     
 ──────────────────────────────────────────────────────
              Arithmetic Intensity (FLOP/byte)

If your kernel is below the roofline:

- Left of the ridge point → memory-bound → optimize memory access
- Right of the ridge point → compute-bound → optimize ALU usage

```

```bash

# Generate roofline with Nsight Compute
ncu --set roofline ./my_app
# Shows where your kernel sits relative to the hardware limits

```

---

## Self-Assessment

### Q1: What is the difference between Nsight Systems and Nsight Compute

**Nsight Systems** (`nsys`) provides a system-wide timeline view — it shows when kernels launch, how long they take, CPU-GPU synchronization points, and data transfers. Use it first to find *which* kernel to optimize.

**Nsight Compute** (`ncu`) provides deep per-kernel analysis — memory throughput, occupancy, instruction mix, bank conflicts, and warp divergence. Use it after identifying the bottleneck kernel to understand *why* it's slow.

### Q2: How do you determine if a kernel is memory-bound or compute-bound

Use the **roofline model** or Nsight Compute metrics. Check two metrics:

- **SM (Compute) Throughput** — percentage of peak compute being used
- **Memory Throughput** — percentage of peak memory bandwidth being used

If memory throughput is high and compute is low → memory-bound (optimize data access). If compute is high and memory is low → compute-bound (optimize arithmetic). If both are low → likely a latency-bound problem (increase occupancy or reduce synchronization).

### Q3: What AMD tools correspond to NVIDIA's profiling tools

| NVIDIA | AMD |
| --- | --- |
| Nsight Systems | `rocprof --stats` / ROCm Trace |
| Nsight Compute | `rocprof` with hardware counters / Omniperf |
| NVTX | `roctx` annotations |
| CUPTI | ROCTracer API |
| Nsight Graphics | Radeon GPU Profiler (RGP) |

---

## Notes

- Always profile with **release builds** (`-O2`/`-O3`) — debug builds have different performance characteristics
- Use `nsys` first (find the bottleneck kernel), then `ncu` (deep-dive into that kernel)
- Nsight Compute's "Baseline" feature lets you compare before/after optimization
- `ncu` has high overhead — profile one kernel launch at a time, not the entire app
- On HPC clusters, use `nsys` with `--trace=cuda,nvtx,osrt` for comprehensive tracing
