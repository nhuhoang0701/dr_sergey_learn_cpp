# Use Intel VTune and AMD uProf for micro-architectural profiling

**Category:** Performance & CPU Architecture  
**Standard:** C++17 (tool-agnostic)  
**Reference:** <https://www.intel.com/content/www/us/en/developer/tools/oneapi/vtune-profiler.html> <https://developer.amd.com/amd-uprof/>  

---

## Topic Overview

### Why Standard Profilers Aren't Enough

Tools like `perf` and `gprof` show **where** time is spent. Micro-architectural profilers show **why** — cache misses, branch mispredictions, port contention, front-end stalls, etc.

### Intel VTune Profiler

VTune provides several analysis types:

| Analysis Type | What It Shows |
| --- | --- |
| **Hotspots** | Functions consuming the most CPU time |
| **Microarchitecture Exploration** | Top-down pipeline analysis (retiring, bad speculation, FE/BE bound) |
| **Memory Access** | Cache miss rates, DRAM bandwidth, NUMA effects |
| **Threading** | Lock contention, thread imbalance, synchronization overhead |
| **HPC Performance** | GFLOPS, vectorization efficiency, roofline model |

**Command-line usage:**

```bash

# Collect microarchitecture data
vtune -collect uarch-exploration -result-dir vtune_results ./my_app

# Generate a report
vtune -report summary -result-dir vtune_results

# Source-level annotation
vtune -report hotspots -result-dir vtune_results -source-object my_app

# Interactive GUI
vtune-gui vtune_results

```

**Typical VTune output (Microarchitecture Exploration):**

```text

Pipeline Slots Analysis:
  Retiring:          32.5%   ← useful work
  Bad Speculation:    8.2%   ← branch mispredictions
  Front-End Bound:    5.1%   ← instruction delivery stalls
  Back-End Bound:    54.2%   ← execution stalls (memory or compute)
    Memory Bound:    41.8%   ← cache misses dominate
    Core Bound:      12.4%   ← port contention

Top Memory Bound Hotspots:
  L1 Bound:    8.3%
  L2 Bound:   12.1%
  L3 Bound:   15.7%
  DRAM Bound:  5.7%

```

### AMD uProf

AMD's profiler for Zen-based processors. Similar capabilities to VTune but optimized for AMD hardware.

```bash

# Collect micro-architecture data
AMDuProfCLI collect --config tbp -o uprof_results ./my_app

# IBS (Instruction Based Sampling) for precise attribution
AMDuProfCLI collect --config ibs -o uprof_results ./my_app

# Generate report
AMDuProfCLI report -i uprof_results

# Launch GUI
AMDuProfUI

```

**AMD uProf analysis types:**

| Analysis | Description |
| --- | --- |
| **Time-based profiling (TBP)** | CPU time per function |
| **Instruction-Based Sampling (IBS)** | Precise µop-level analysis |
| **L3 Cache Access** | Cache miss patterns |
| **Memory Bandwidth** | Channel utilization |

### Roofline Model Analysis

Both tools support **roofline model** visualization:

```cpp

Performance (GFLOPS/s)
     ^
     |        * Peak Compute  _______________
     |       /               /
     |      /               /
     |     /  * Your code  /
     |    /               /
     |   /  Memory bound / Compute bound
     |  /               /
     +----------------------------------------------------->
        Arithmetic Intensity (FLOPS/byte)

```

If your code falls below the roofline, the gap shows your optimization potential.

### Integrating with C++ Code

```cpp

#include <ittnotify.h>  // VTune instrumentation API

void process_batch(const Data& data) {
    // Mark a region for VTune analysis
    __itt_domain* domain = __itt_domain_create("MyApp");
    __itt_string_handle* task = __itt_string_handle_create("process_batch");

    __itt_task_begin(domain, __itt_null, __itt_null, task);

    // ... hot code to analyze ...

    __itt_task_end(domain);
}

```

```cmake

# Link VTune instrumentation API
find_package(ittnotify REQUIRED)
target_link_libraries(myapp PRIVATE ittnotify)

```

---

## Self-Assessment

### Q1: When would you use VTune's "Memory Access" analysis

When `perf stat` or VTune's microarchitecture exploration shows high **Back-End Bound → Memory Bound** percentage. The Memory Access analysis drills down to:

- Which cache level is the bottleneck (L1, L2, L3, DRAM).
- Which data structures cause cache misses.
- NUMA remote access patterns in multi-socket systems.

### Q2: What is IBS and why is it more accurate than sampling

**Instruction-Based Sampling (IBS)** is an AMD hardware feature that:

- Tags a specific µop as it enters the pipeline.
- Tracks that µop through the entire pipeline (fetch, decode, dispatch, execute, retire).
- Records precise events: cache levels hit/miss, branch taken/not-taken, latency.

Unlike PMU sampling (which attributes events to nearby instructions), IBS attributes events to the **exact instruction** that caused them — no skid.

### Q3: How do you use the roofline model to guide optimization

1. **Measure** your code's arithmetic intensity (FLOPS / bytes transferred).
2. **Plot** it on the roofline: if you're below the sloped line, you're **memory bound**; if below the flat ceiling, you're **compute bound**.
3. **Optimize accordingly:**
   - Memory bound → improve data locality, use prefetching, reduce data size.
   - Compute bound → use SIMD, reduce instruction count, improve ILP.

---

## Notes

- VTune is free for all users (part of Intel oneAPI toolkit).
- AMD uProf is free and works on all AMD Zen processors.
- Both tools require root/admin access for hardware counter access.
- Use `likwid` as an open-source alternative that works on both Intel and AMD.
- Always profile on the **target hardware** — results differ significantly between CPU generations.
