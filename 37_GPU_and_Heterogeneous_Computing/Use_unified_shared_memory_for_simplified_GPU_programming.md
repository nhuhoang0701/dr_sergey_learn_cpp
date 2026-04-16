# Use Unified Shared Memory for Simplified GPU Programming

**Category:** GPU & Heterogeneous Computing  
**Standard:** C++17 / CUDA 12.x / SYCL 2020  
**Reference:** https://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#unified-memory-programming  

---

## Topic Overview

Unified Shared Memory (USM) unifies the host and device address spaces so that a single pointer is valid on both CPU and GPU. This eliminates explicit `memcpy` calls for many workloads and dramatically simplifies code—especially for complex, pointer-rich data structures like linked lists, trees, and graphs that are difficult or impossible to marshal across address spaces manually.

CUDA calls this "Managed Memory" (`cudaMallocManaged`), while SYCL 2020 defines three USM allocation types: `malloc_device` (device-only), `malloc_host` (host-allocated, device-accessible via PCIe), and `malloc_shared` (automatically migrated). The key performance insight is that shared/managed memory relies on page-fault-driven migration, which introduces latency on first access. Prefetching (`cudaMemPrefetchAsync` / `sycl::queue::prefetch`) moves pages proactively, and memory advise hints (`cudaMemAdvise`) inform the runtime about access patterns (read-mostly, preferred location) to optimize placement.

Migration happens at page granularity (typically 4 KB on CPU, 64 KB on GPU). Thrashing occurs when both CPU and GPU frequently access the same pages within a short time window—leading to continuous page migration and severe performance degradation. The solution is to structure access patterns so that CPU and GPU touch disjoint page ranges, or synchronize access phases clearly.

| USM Type          | CUDA API              | SYCL API                | Host R/W | Device R/W | Migration         |
| --- | --- | --- | --- | --- | --- |
| Device memory     | `cudaMalloc`          | `sycl::malloc_device`   | No       | Yes        | None              |
| Host memory       | `cudaMallocHost`      | `sycl::malloc_host`     | Yes      | Via PCIe   | None              |
| Shared/Managed    | `cudaMallocManaged`   | `sycl::malloc_shared`   | Yes      | Yes        | On-demand / hint  |

```cpp

Page Migration Flow (cudaMallocManaged):

 CPU accesses page → Page fault on GPU  → Driver migrates page to CPU
 GPU accesses page → Page fault on CPU  → Driver migrates page to GPU

 With prefetch:
 cudaMemPrefetchAsync(ptr, size, device) → Bulk migration, no faults
                                          → ~10× lower first-access latency

```

---

## Self-Assessment

### Q1: Demonstrate CUDA managed memory with RAII, and show the impact of prefetch and memory advise

```cpp

#include <cuda_runtime.h>
#include <cstdio>
#include <memory>
#include <chrono>

#define CUDA_CHECK(call) do { \
    cudaError_t e = (call); \
    if (e != cudaSuccess) { \
        fprintf(stderr, "CUDA error: %s at %s:%d\n", \
                cudaGetErrorString(e), __FILE__, __LINE__); \
        exit(1); } \
    } while(0)

// RAII wrapper for managed memory
template <typename T>
struct ManagedDeleter {
    void operator()(T* p) const { cudaFree(p); }
};

template <typename T>
using ManagedPtr = std::unique_ptr<T[], ManagedDeleter<T>>;

template <typename T>
ManagedPtr<T> make_managed(size_t n) {
    T* ptr;
    CUDA_CHECK(cudaMallocManaged(&ptr, n * sizeof(T)));
    return ManagedPtr<T>(ptr);
}

__global__ void saxpy(float* y, const float* x, float a, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) y[i] = a * x[i] + y[i];
}

double run_saxpy(float* x, float* y, int n, const char* label) {
    auto t0 = std::chrono::high_resolution_clock::now();
    saxpy<<<(n + 255) / 256, 256>>>(y, x, 2.0f, n);
    CUDA_CHECK(cudaDeviceSynchronize());
    auto t1 = std::chrono::high_resolution_clock::now();
    double ms = std::chrono::duration<double, std::milli>(t1 - t0).count();
    printf("%-35s %8.3f ms\n", label, ms);
    return ms;
}

int main() {
    constexpr int N = 1 << 24;
    int dev = 0;
    CUDA_CHECK(cudaSetDevice(dev));

    auto x = make_managed<float>(N);
    auto y = make_managed<float>(N);

    // Initialize on CPU (pages reside on CPU after this)
    for (int i = 0; i < N; ++i) {
        x[i] = 1.0f;
        y[i] = 2.0f;
    }

    // Run 1: Cold — pages migrate on-demand via page faults
    run_saxpy(x.get(), y.get(), N, "No prefetch (page faults):");

    // Re-init on CPU
    for (int i = 0; i < N; ++i) { x[i] = 1.0f; y[i] = 2.0f; }

    // Run 2: Prefetch to GPU before kernel
    CUDA_CHECK(cudaMemPrefetchAsync(x.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaMemPrefetchAsync(y.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaDeviceSynchronize());
    run_saxpy(x.get(), y.get(), N, "With prefetch:");

    // Run 3: Advise read-mostly for x (replicated on both)
    for (int i = 0; i < N; ++i) { x[i] = 1.0f; y[i] = 2.0f; }
    CUDA_CHECK(cudaMemAdvise(x.get(), N * sizeof(float),
                              cudaMemAdviseSetReadMostly, dev));
    CUDA_CHECK(cudaMemPrefetchAsync(x.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaMemPrefetchAsync(y.get(), N * sizeof(float), dev));
    CUDA_CHECK(cudaDeviceSynchronize());
    run_saxpy(x.get(), y.get(), N, "ReadMostly(x) + prefetch:");

    // Verify on CPU (pages migrate back automatically)
    printf("y[0] = %f (expected 4.0)\n", y[0]);

    return 0;
}

```

### Q2: Show USM thrashing and how to eliminate it with phased access patterns

```cpp

#include <cuda_runtime.h>
#include <cstdio>
#include <chrono>

#define CUDA_CHECK(x) do { cudaError_t e=(x); \
    if(e!=cudaSuccess){printf("CUDA err %d\n",e);exit(1);}}while(0)

__global__ void gpu_write(float* data, int n) {
    int i = blockIdx.x * blockDim.x + threadIdx.x;
    if (i < n) data[i] = static_cast<float>(i);
}

void cpu_read(const float* data, int n) {
    volatile float sum = 0;
    for (int i = 0; i < n; i += 1024) sum += data[i];
}

int main() {
    constexpr int N = 1 << 20;
    int dev = 0;
    float* data;
    CUDA_CHECK(cudaMallocManaged(&data, N * sizeof(float)));

    // === BAD: Interleaved CPU/GPU access causes thrashing ===
    auto t0 = std::chrono::high_resolution_clock::now();
    for (int iter = 0; iter < 10; ++iter) {
        gpu_write<<<(N+255)/256, 256>>>(data, N);
        CUDA_CHECK(cudaDeviceSynchronize());
        cpu_read(data, N);  // Forces page migration GPU→CPU
        // Next iteration: GPU→CPU migration again
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    double thrash_ms = std::chrono::duration<double,
                       std::milli>(t1 - t0).count();

    // === GOOD: Phased access — batch GPU work, then batch CPU ===
    t0 = std::chrono::high_resolution_clock::now();

    // Phase 1: All GPU work
    for (int iter = 0; iter < 10; ++iter) {
        gpu_write<<<(N+255)/256, 256>>>(data, N);
    }
    CUDA_CHECK(cudaDeviceSynchronize());

    // Phase 2: Prefetch to CPU, then all CPU reads
    CUDA_CHECK(cudaMemPrefetchAsync(data, N * sizeof(float),
                                     cudaCpuDeviceId));
    CUDA_CHECK(cudaDeviceSynchronize());
    for (int iter = 0; iter < 10; ++iter) {
        cpu_read(data, N);
    }

    t1 = std::chrono::high_resolution_clock::now();
    double phased_ms = std::chrono::duration<double,
                       std::milli>(t1 - t0).count();

    printf("Thrashing pattern:  %8.2f ms\n", thrash_ms);
    printf("Phased pattern:     %8.2f ms\n", phased_ms);
    printf("Speedup:            %.1fx\n", thrash_ms / phased_ms);

    CUDA_CHECK(cudaFree(data));
    return 0;
}

```

### Q3: Implement the same algorithm using SYCL USM types and compare malloc_shared vs malloc_device + explicit copy

```cpp

#include <sycl/sycl.hpp>
#include <iostream>
#include <chrono>

template <typename F>
double bench(F&& f) {
    auto t0 = std::chrono::high_resolution_clock::now();
    f();
    auto t1 = std::chrono::high_resolution_clock::now();
    return std::chrono::duration<double, std::milli>(t1 - t0).count();
}

int main() {
    constexpr size_t N = 1 << 22;
    sycl::queue q{sycl::gpu_selector_v};

    // === Approach 1: malloc_shared (automatic migration) ===
    float* shared = sycl::malloc_shared<float>(N, q);
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;

    double t_shared_cold = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            shared[i] *= 2.0f;
        }).wait();
    });

    // Reset and prefetch
    for (size_t i = 0; i < N; ++i) shared[i] = 1.0f;
    q.prefetch(shared, N * sizeof(float)).wait();

    double t_shared_warm = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            shared[i] *= 2.0f;
        }).wait();
    });

    // === Approach 2: malloc_device + explicit copy ===
    float* dev = sycl::malloc_device<float>(N, q);
    std::vector<float> host(N, 1.0f);

    double t_explicit = bench([&] {
        q.memcpy(dev, host.data(), N * sizeof(float)).wait();
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            dev[i] *= 2.0f;
        }).wait();
        q.memcpy(host.data(), dev, N * sizeof(float)).wait();
    });

    // === Approach 3: malloc_host (zero-copy, CPU-resident) ===
    float* h_usm = sycl::malloc_host<float>(N, q);
    for (size_t i = 0; i < N; ++i) h_usm[i] = 1.0f;

    double t_host_usm = bench([&] {
        q.parallel_for(sycl::range<1>(N), [=](sycl::id<1> i) {
            h_usm[i] *= 2.0f;
        }).wait();
    });

    printf("%-35s %8.2f ms\n", "malloc_shared (cold):", t_shared_cold);
    printf("%-35s %8.2f ms\n", "malloc_shared (prefetched):",t_shared_warm);
    printf("%-35s %8.2f ms\n", "malloc_device + copy:", t_explicit);
    printf("%-35s %8.2f ms\n", "malloc_host (zero-copy):", t_host_usm);

    /*
     When to use each:
     ┌────────────────────┬──────────────────────────────┐
     │ USM type           │ Best for                     │
     ├────────────────────┼──────────────────────────────┤
     │ malloc_shared      │ Prototyping, complex structs │
     │ malloc_device      │ Max performance, bulk data   │
     │ malloc_host        │ Small, frequently read by GPU│
     └────────────────────┴──────────────────────────────┘
    */

    sycl::free(shared, q);
    sycl::free(dev, q);
    sycl::free(h_usm, q);
    return 0;
}

```

---

## Notes

- Managed/shared memory migration granularity is 64 KB on most NVIDIA GPUs — even touching one float migrates the entire 64 KB page.
- `cudaMemAdviseSetPreferredLocation` pins pages to a specific processor, reducing migration but potentially causing remote access latency.
- `cudaMemAdviseSetAccessedBy` creates direct mappings to avoid migration entirely — pages stay in place but are accessed remotely via NVLink/PCIe.
- SYCL `malloc_shared` portability varies — Intel GPUs support true shared virtual memory, while NVIDIA backend maps to CUDA managed memory.
- For pointer-rich data structures (trees, graphs), USM is often the only viable option — explicit marshaling of pointer graphs across address spaces is error-prone and impractical.
- Over-subscription (allocating more managed memory than GPU VRAM) works on modern CUDA but triggers eviction/migration — profile with `nvprof` or `nsys` to detect.
