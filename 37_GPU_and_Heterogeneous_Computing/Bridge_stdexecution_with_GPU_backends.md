# Bridge std::execution with GPU Backends

**Category:** GPU & Heterogeneous Computing  
**Standard:** C++26 (P2300 std::execution) / C++20  
**Reference:** https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2024/p2300r10.html  

---

## Topic Overview

P2300 (`std::execution`, formerly "Senders/Receivers") introduces a structured, composable framework for asynchronous and parallel work in C++. Its key abstraction—schedulers, senders, and receivers—naturally extends to GPU offloading: a GPU scheduler creates senders that represent work to be executed on a GPU, and `when_all` / `then` combinators compose mixed CPU/GPU pipelines without manual synchronization.

NVIDIA's `nvexec` library (part of stdexec, the reference implementation) provides GPU schedulers backed by CUDA streams. A `nvexec::stream_scheduler` submits senders to a CUDA stream, enabling kernels expressed as lambdas to be composed into sender chains. The runtime manages memory transfers and synchronization, abstracting away `cudaMemcpyAsync` and `cudaStreamSynchronize`. This lets developers write heterogeneous pipelines that read like sequential code but execute concurrently across CPU and GPU.

The critical insight is that `std::execution` separates **what** to compute from **where** to compute it. The same sender chain can target different schedulers (thread pool, GPU, FPGA) without changing business logic—a powerful separation of concerns for large-scale heterogeneous applications.

```cpp

Sender/Receiver Pipeline with GPU:

  schedule(cpu_sched)           schedule(gpu_sched)
       │                              │
       ▼                              ▼
  then(prepare_data)            then(gpu_kernel)
       │                              │
       └──────── when_all ────────────┘
                    │
                    ▼
              then(postprocess)
                    │
                    ▼
              sync_wait(result)

```

| Concept        | Role in CPU context      | Role in GPU context                |
| --- | --- | --- |
| Scheduler      | Thread pool              | CUDA stream / SYCL queue            |
| Sender         | Async task description   | GPU kernel + transfer description   |
| Receiver       | Completion handler       | Host callback on kernel completion  |
| `then`         | Continuation on CPU      | Chain kernels or CPU post-processing|
| `when_all`     | Join CPU tasks           | Join CPU + GPU tasks                |
| `transfer`     | Move across thread pools | Move work from CPU to GPU scheduler |

---

## Self-Assessment

### Q1: Write a basic stdexec pipeline that schedules work on a GPU using nvexec::stream_scheduler

```cpp

// Requires: stdexec + nvexec (NVIDIA's std::execution GPU backend)
// Build: nvcc -std=c++20 -I<stdexec>/include -I<nvexec>/include

#include <stdexec/execution.hpp>
#include <nvexec/stream_context.cuh>
#include <cstdio>

// A simple GPU "kernel" expressed as a sender chain
int main() {
    // Create a GPU stream context — owns a CUDA stream
    nvexec::stream_context gpu_ctx{};
    auto gpu_sched = gpu_ctx.get_scheduler();

    // Build a sender pipeline
    auto work = stdexec::schedule(gpu_sched)
        | stdexec::then([] __host__ __device__ () {
            // Runs on GPU
            return 42;
        })
        | stdexec::then([] __host__ __device__ (int val) {
            // Also runs on GPU (same stream)
            return val * 2;
        });

    // Synchronously wait for the result on the host
    auto [result] = stdexec::sync_wait(std::move(work)).value();

    printf("Result: %d\n", result);  // 84
    return 0;
}

```

### Q2: Compose a mixed CPU/GPU pipeline using when_all to join heterogeneous tasks and transfer between schedulers

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <nvexec/stream_context.cuh>
#include <vector>
#include <numeric>
#include <cstdio>

// Simulates a CPU-heavy preprocessing step
auto cpu_preprocess(exec::static_thread_pool::scheduler cpu_sched,
                    std::vector<float>& data) {
    return stdexec::schedule(cpu_sched)
        | stdexec::then([&data]() {
            // CPU work: fill data (e.g., decode, parse, normalize)
            std::iota(data.begin(), data.end(), 0.0f);
            printf("[CPU] Preprocessed %zu elements\n", data.size());
            return data.data();
        });
}

// GPU computation as a sender
auto gpu_compute(nvexec::stream_context::scheduler_t gpu_sched,
                 float* d_ptr, size_t n) {
    return stdexec::schedule(gpu_sched)
        | stdexec::then([d_ptr, n] __host__ __device__ () {
            // In real code this would be a parallel_for or kernel launch
            // nvexec bulk() enables parallel GPU execution
            return d_ptr;
        });
}

int main() {
    constexpr size_t N = 1 << 16;

    // CPU scheduler (thread pool)
    exec::static_thread_pool cpu_pool{4};
    auto cpu_sched = cpu_pool.get_scheduler();

    // GPU scheduler
    nvexec::stream_context gpu_ctx{};
    auto gpu_sched = gpu_ctx.get_scheduler();

    std::vector<float> host_data(N);

    // Build a pipeline: CPU prep + GPU compute in parallel, then merge
    auto pipeline =
        stdexec::when_all(
            // Branch 1: CPU preprocessing
            cpu_preprocess(cpu_sched, host_data)
                | stdexec::then([](float* ptr) {
                    printf("[CPU] Data ready at %p\n",
                           static_cast<void*>(ptr));
                    return ptr;
                }),

            // Branch 2: Independent GPU warmup
            stdexec::schedule(gpu_sched)
                | stdexec::then([] __host__ __device__ () {
                    // GPU warmup / context init
                    return 0;
                })
        )
        | stdexec::then([](float* ptr, int) {
            printf("[Host] Both branches completed. Merging.\n");
            return ptr;
        });

    auto [result_ptr] = stdexec::sync_wait(std::move(pipeline)).value();
    printf("Pipeline done. First element: %f\n", host_data[0]);

    return 0;
}

```

### Q3: Implement a bulk GPU operation using stdexec::bulk with a GPU scheduler, equivalent to a CUDA parallel_for

```cpp

#include <stdexec/execution.hpp>
#include <nvexec/stream_context.cuh>
#include <cstdio>
#include <cmath>

// nvexec maps bulk() to CUDA kernel launches
int main() {
    constexpr int N = 1 << 20;

    nvexec::stream_context gpu_ctx{};
    auto gpu_sched = gpu_ctx.get_scheduler();

    // Allocate unified memory (accessible from sender lambdas)
    float* a;
    float* b;
    float* c;
    cudaMallocManaged(&a, N * sizeof(float));
    cudaMallocManaged(&b, N * sizeof(float));
    cudaMallocManaged(&c, N * sizeof(float));

    // Initialize on host
    for (int i = 0; i < N; ++i) {
        a[i] = static_cast<float>(i);
        b[i] = static_cast<float>(i) * 0.5f;
    }

    // Build sender chain with bulk for GPU parallel execution
    auto work = stdexec::schedule(gpu_sched)
        // bulk(count, func) — launches count parallel invocations
        | stdexec::bulk(N,
            [a, b, c] __host__ __device__ (int i) {
                c[i] = a[i] + b[i];
            })
        | stdexec::then([c, N] __host__ __device__ () {
            // Verification step (runs after bulk completes)
            // In device context with unified memory
            return c[0];
        });

    auto [first] = stdexec::sync_wait(std::move(work)).value();
    printf("c[0] = %f (expected 0.0)\n", first);
    printf("c[1] = %f (expected 1.5)\n", c[1]);

    cudaFree(a);
    cudaFree(b);
    cudaFree(c);
    return 0;
}

/*
 Execution model mapping:

 stdexec concept  │  CUDA equivalent
 ─────────────────┼───────────────────
 schedule(gpu)    │  Select CUDA stream
 bulk(N, fn)      │  kernel<<<...>>>(fn) with N threads
 then(fn)         │  Subsequent kernel or host callback
 when_all(a, b)   │  cudaStreamWaitEvent across streams
 sync_wait(s)     │  cudaStreamSynchronize

 Key advantage: Composability.
 A bulk sender can be a node in a larger DAG that includes
 CPU tasks, I/O, and multiple GPU devices — all expressed
 as a single sender expression.
*/

```

---

## Notes

- `stdexec` (P2300) is targeting C++26; use the reference implementation at `github.com/NVIDIA/stdexec` today.
- `nvexec::stream_scheduler` maps sender operations to CUDA stream operations — `then` chains kernels, `when_all` inserts stream events.
- `bulk(N, fn)` on a GPU scheduler launches a CUDA kernel with N logical threads — the runtime selects grid/block dimensions.
- `transfer(scheduler)` moves a sender's continuation to a different execution context — use it to bounce between CPU and GPU.
- Error propagation through `set_error` channel replaces manual `cudaGetLastError()` checks — errors flow through the sender chain.
- The separation of scheduling policy from algorithm logic makes GPU code testable on CPU by swapping the scheduler.
