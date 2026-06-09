# Profile GPU kernels with NSight and ROCProfiler

**Category:** GPU and Heterogeneous Computing  
**Standard:** C++17  
**Reference:** <https://developer.nvidia.com/nsight-compute>  

---

## Topic Overview

### Why GPU Profiling Is Different

If you try to use a CPU profiler like `perf` or VTune to analyze a GPU kernel, you'll get almost nothing useful. GPU execution is a fundamentally different animal, and it needs its own specialized tools. Here's why:

- GPU execution is **asynchronous** - the kernel launch call returns on the CPU before the GPU has done any work, so a kernel launch does not equal kernel execution
- Thousands of threads run simultaneously - the bottlenecks you care about (warp divergence, memory coalescing, occupancy) simply don't exist on CPUs
- The memory hierarchy is completely different - shared memory, L1/L2 caches, coalescing behavior, and bank conflicts all need GPU-specific insight
- Concepts like occupancy, warp efficiency, and instruction throughput are GPU-only ideas

### NVIDIA Profiling Tool Stack

There are a few tools in the NVIDIA ecosystem, and it's worth knowing which one to reach for when. If the table feels like a lot, the short version is: use `nsys` first to find the slow spot, then use `ncu` to understand why it's slow.

| Tool | Purpose | Level |
| --- | --- | --- |
| **Nsight Systems** (`nsys`) | System-wide timeline: CPU+GPU, API calls, kernel launches | High-level |
| **Nsight Compute** (`ncu`) | Per-kernel deep analysis: memory, compute, occupancy | Low-level |
| **CUPTI** | Programmatic profiling API | Library |
| **nvprof** (legacy) | Deprecated - use nsys/ncu instead | - |

### Nsight Systems — Finding the Right Kernel to Optimize

The first step in any GPU optimization workflow is figuring out *which* kernel is even worth your time. Nsight Systems (`nsys`) gives you a system-wide timeline showing CPU activity, GPU activity, kernel launches, and data transfers all lined up together. Run it like this:

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

The GUI timeline view shows you both the CPU thread and the GPU stream side by side so you can immediately spot wasted time:

```cpp
CPU Thread 0: [cudaMalloc][cudaMemcpy][<<<kernel_A>>>][<<<kernel_B>>>][cudaMemcpy]
GPU Stream 0:              [memcpy H2D] [kernel_A ████████][kernel_B ██][memcpy D2H]
                            idle ^           ^ target for optimization
```

Notice that the GPU sits idle while the CPU is doing setup work. That gap is a signal - you want to minimize idle time on both sides.

### Nsight Compute — Deep Kernel Analysis

Once you've identified a slow kernel with Nsight Systems, you switch to Nsight Compute (`ncu`) for the deep dive. This tool gives you detailed counters about what's happening *inside* the kernel - memory efficiency, occupancy, instruction mix, and more.

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

The `--diff` flag is particularly valuable - it shows you exactly how much each metric changed between two versions of your kernel, so you can confirm that your optimization actually helped.

### Key Nsight Compute Metrics Explained

Here's an example of what the Nsight Compute output looks like for a matrix multiplication kernel. Each number tells a story:

```cpp
┌─────────────────────────────────────────────┐
│ Kernel: matrix_mul  Grid: [128,128,1]       │
│ Block: [32,32,1]    Duration: 1.2 ms        │
├─────────────────────────────────────────────┤
│ Compute (SM) Throughput:        45.2%       │ <- Not compute-bound
│ Memory Throughput:              87.3%       │ <- Memory-bound!
│ Achieved Occupancy:             75.0%       │ <- Good (>50%)
│                                             │
│ Global Load Efficiency:         25.0%       │ <- BAD! Uncoalesced
│ Shared Memory Bank Conflicts:   1.2M/s     │ <- Needs padding fix
│ Warp Execution Efficiency:      62.5%       │ <- Branch divergence
│ Register Spills (stores):       8/thread    │ <- Register pressure
└─────────────────────────────────────────────┘
```

Reading these numbers together tells you what to fix and in what order:

- **Memory-bound** (87.3% memory throughput) - the bottleneck is memory access, not arithmetic; optimize memory access patterns first
- **Low global load efficiency** (25%) - threads in a warp are not reading consecutive addresses, which means the GPU is fetching 4x more data than it needs; switch to a Structure of Arrays layout
- **Bank conflicts** - threads in a warp are hitting the same shared memory bank; add padding to your shared memory arrays
- **Low warp efficiency** (62.5%) - threads in a warp are taking different code paths; reduce branch divergence

### AMD ROCProfiler for AMD GPUs

AMD provides analogous profiling tools through the ROCm stack. The workflow is similar: gather timing stats first, then drill into hardware counters for specific kernels.

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
# SQ_WAVES          - total wavefronts launched
# SQ_INSTS_VALU     - vector ALU instructions
# SQ_INSTS_SMEM     - scalar memory instructions
# TA_TOTAL_WAVEFRONTS_SENT - texture/memory wavefronts
# TCP_TCC_READ_REQ  - L2 cache read requests
```

### AMD Omniperf — High-Level Analysis

Omniperf sits on top of ROCProfiler and provides a higher-level analysis report, similar to what Nsight Compute's guided analysis mode gives you. It's your go-to for roofline analysis and memory utilization breakdowns on AMD hardware.

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

Instead of profiling a whole application, you can annotate specific sections of your code so the profiler knows which regions to highlight. NVTX markers show up as colored ranges on the Nsight Systems timeline, and CUDA Events give you precise kernel timing directly from your C++ code.

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

CUDA Events are more accurate than host-side timers because they measure time on the GPU's own clock, without any host-device synchronization noise.

### The Roofline Model

The roofline model is the single most useful mental framework for GPU optimization. It tells you definitively whether your kernel is limited by arithmetic throughput or by memory bandwidth - which determines what you should try to fix.

```cpp
          Peak Compute ─────────────────────────────────
         /
        /     <- Compute-bound region
       /
Peak ─/──────── Your kernel (*)
     /
    /  <- Memory-bound region
   /
  /
 ──────────────────────────────────────────────────────
              Arithmetic Intensity (FLOP/byte)

If your kernel is below the roofline:

- Left of the ridge point -> memory-bound -> optimize memory access
- Right of the ridge point -> compute-bound -> optimize ALU usage
```

The x-axis is arithmetic intensity: how many floating-point operations you do per byte of memory you transfer. The y-axis is performance. Your kernel plots as a point, and the shape of the roofline tells you where the ceiling is. If you're on the left side (low arithmetic intensity), buying a faster GPU won't help - you need to reduce memory traffic.

```bash
# Generate roofline with Nsight Compute
ncu --set roofline ./my_app
# Shows where your kernel sits relative to the hardware limits
```

---

## Self-Assessment

### Q1: What is the difference between Nsight Systems and Nsight Compute

**Nsight Systems** (`nsys`) provides a system-wide timeline view - it shows when kernels launch, how long they take, CPU-GPU synchronization points, and data transfers. Use it first to find *which* kernel to optimize.

**Nsight Compute** (`ncu`) provides deep per-kernel analysis - memory throughput, occupancy, instruction mix, bank conflicts, and warp divergence. Use it after identifying the bottleneck kernel to understand *why* it's slow.

### Q2: How do you determine if a kernel is memory-bound or compute-bound

Use the **roofline model** or Nsight Compute metrics. Check two metrics:

- **SM (Compute) Throughput** - percentage of peak compute being used
- **Memory Throughput** - percentage of peak memory bandwidth being used

If memory throughput is high and compute is low -> memory-bound (optimize data access). If compute is high and memory is low -> compute-bound (optimize arithmetic). If both are low -> likely a latency-bound problem (increase occupancy or reduce synchronization).

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

- Always profile with **release builds** (`-O2`/`-O3`) - debug builds have different performance characteristics and will mislead you.
- Use `nsys` first to find the bottleneck kernel, then `ncu` to deep-dive into that kernel - this two-step workflow saves a lot of time.
- Nsight Compute's "Baseline" feature lets you compare before/after optimization so you can quantify the impact of each change.
- `ncu` has high overhead - profile one kernel launch at a time, not the entire application, or you'll wait a very long time.
- On HPC clusters, use `nsys` with `--trace=cuda,nvtx,osrt` for comprehensive tracing that captures the full picture.
