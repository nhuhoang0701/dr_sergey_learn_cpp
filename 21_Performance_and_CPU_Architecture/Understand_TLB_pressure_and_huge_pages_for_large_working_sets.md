# Understand TLB pressure and huge pages for large working sets

**Category:** Performance & CPU Architecture  
**Item:** #636  
**Standard:** C++17  
**Reference:** <https://www.kernel.org/doc/html/latest/admin-guide/mm/transhuge.html>  

---

## Topic Overview

The Translation Lookaside Buffer (TLB) caches virtual-to-physical page mappings. With 4KB pages and a working set of 1GB, you need 262,144 pages — far more than the TLB can hold (~1536 entries). Huge pages (2MB/1GB) reduce the page count by 512x/262144x.

```cpp

4KB pages (default):                2MB huge pages:
  Working set: 1 GB                   Working set: 1 GB
  Pages: 262,144                      Pages: 512
  TLB entries: ~1,536                 TLB entries: ~1,536
  Coverage: 6 MB                      Coverage: 3 GB
  -> constant TLB misses!             -> everything fits in TLB!

TLB miss penalty: ~7-100 cycles (page walk through 4 levels)

```

| Page size | TLB entries (typ.) | Coverage | Use case |
| --- | --- | --- | --- |
| 4 KB | 1536 (L1+L2) | ~6 MB | Default, small programs |
| 2 MB | 32 (dedicated) | 64 MB | Large arrays, databases |
| 1 GB | 4 (dedicated) | 4 GB | HPC, in-memory DBs |

---

## Self-Assessment

### Q1: How TLB misses add latency

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <random>
#include <numeric>

// TLB miss triggers a PAGE WALK:
// 1. CPU reads PML4 table entry    (~4 cycles, L1 hit)
// 2. CPU reads PDPT entry           (~4 cycles)
// 3. CPU reads Page Directory entry  (~4 cycles)
// 4. CPU reads Page Table entry      (~4 cycles if cached, ~200 if DRAM)
// Total: 7-100+ cycles per TLB miss
//
// x86-64 TLB hierarchy (Intel Skylake):
// L1 dTLB: 64 entries (4KB), 32 entries (2MB)
// L2 sTLB: 1536 entries (4KB/2MB shared)

int main() {
    // Demonstrate TLB pressure with random access across many pages
    std::vector<size_t> sizes = {
        1ULL << 20,    // 1 MB = 256 pages
        16ULL << 20,   // 16 MB = 4096 pages (exceeds TLB)
        256ULL << 20,  // 256 MB = 65536 pages
        1ULL << 30     // 1 GB = 262144 pages
    };

    for (size_t bytes : sizes) {
        size_t n = bytes / sizeof(int);
        std::vector<int> data(n, 1);

        // Random access pattern: touches many pages
        std::mt19937 rng(42);
        std::vector<size_t> indices(1'000'000);
        for (auto& idx : indices) idx = rng() % n;

        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (size_t idx : indices) {
            sum += data[idx];  // random access -> TLB miss likely
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count()
                    / 1'000'000.0;

        std::cout << (bytes >> 20) << " MB: " << ns << " ns/access\n";
    }
    // Expected:
    //   1 MB:   ~5 ns   (fits in TLB, L1/L2 dTLB covers 256 pages)
    //  16 MB:  ~15 ns   (exceeds TLB, page walks start)
    // 256 MB:  ~30 ns   (heavy TLB misses + L3 cache misses)
    //   1 GB:  ~80 ns   (constant TLB misses + DRAM access)
}

```

### Q2: Enable huge pages with `madvise`

```cpp

#include <iostream>
#include <cstring>
#include <chrono>
#include <random>
#include <sys/mman.h>  // Linux only

// Allocate with huge page support on Linux:
void* alloc_huge(size_t bytes) {
    // Method 1: mmap + madvise (Transparent Huge Pages)
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    madvise(ptr, bytes, MADV_HUGEPAGE);  // request THP
    return ptr;
}

void* alloc_normal(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    if (ptr == MAP_FAILED) return nullptr;
    madvise(ptr, bytes, MADV_NOHUGEPAGE);  // force 4KB pages
    return ptr;
}

int main() {
    constexpr size_t SIZE = 256ULL << 20; // 256 MB
    constexpr int ITERS = 10'000'000;

    auto bench = [&](void* mem, const char* label) {
        int* data = static_cast<int*>(mem);
        size_t n = SIZE / sizeof(int);
        std::memset(data, 1, SIZE); // fault all pages

        std::mt19937 rng(42);
        volatile long long sum = 0;
        auto t0 = std::chrono::high_resolution_clock::now();
        for (int i = 0; i < ITERS; ++i) {
            sum += data[rng() % n];
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        double ns = std::chrono::duration_cast<std::chrono::nanoseconds>(t1 - t0).count()
                    / static_cast<double>(ITERS);
        std::cout << label << ": " << ns << " ns/access\n";
    };

    void* normal = alloc_normal(SIZE);
    void* huge = alloc_huge(SIZE);

    if (normal && huge) {
        bench(normal, "4KB pages  ");
        bench(huge,   "2MB pages  ");
        // Typical (256MB, random access):
        // 4KB pages: ~30 ns/access
        // 2MB pages: ~20 ns/access (33% faster, fewer TLB misses)
        munmap(normal, SIZE);
        munmap(huge, SIZE);
    }

    // Measure with perf:
    //   perf stat -e dTLB-load-misses,dTLB-store-misses ./test
    //   4KB: dTLB-load-misses = 5,000,000
    //   2MB: dTLB-load-misses =    50,000  (100x reduction!)
}

```

### Q3: THP vs explicit `MAP_HUGETLB`

```cpp

#include <iostream>
#include <sys/mman.h>
#include <cstring>

// Method 1: Transparent Huge Pages (THP)
// - OS automatically promotes 4KB pages to 2MB when possible
// - No special privileges needed
// - May not always use huge pages (fragmentation)
void* alloc_thp(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS, -1, 0);
    madvise(ptr, bytes, MADV_HUGEPAGE);
    return ptr;
    // Pros: easy, automatic, works with malloc if THP enabled
    // Cons: may silently fall back to 4KB, allocation latency spikes
}

// Method 2: Explicit huge pages (MAP_HUGETLB)
// - Requires preallocated huge page pool:
//   echo 128 > /proc/sys/vm/nr_hugepages  (reserves 128 * 2MB = 256MB)
// - Guaranteed 2MB pages (or allocation fails)
void* alloc_explicit_huge(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB, -1, 0);
    // Returns MAP_FAILED if huge pages not available
    return ptr;
    // Pros: guaranteed huge pages, no fragmentation issues
    // Cons: needs root/privileges, wastes memory if partially used
}

// Method 3: 1GB huge pages (for very large allocations)
void* alloc_1gb_huge(size_t bytes) {
    void* ptr = mmap(nullptr, bytes, PROT_READ | PROT_WRITE,
                     MAP_PRIVATE | MAP_ANONYMOUS | MAP_HUGETLB
                     | (30 << MAP_HUGE_SHIFT), -1, 0);
    // 30 = log2(1GB), requires boot-time reservation:
    //   hugepagesz=1G hugepages=4 on kernel cmdline
    return ptr;
}

int main() {
    // Comparison table:
    std::cout << "Feature comparison:\n";
    std::cout << "+------------------+-------------------+-------------------+\n";
    std::cout << "| Feature          | THP               | MAP_HUGETLB       |\n";
    std::cout << "+------------------+-------------------+-------------------+\n";
    std::cout << "| Page size        | 2MB (auto)        | 2MB or 1GB        |\n";
    std::cout << "| Guaranteed       | No (best effort)  | Yes               |\n";
    std::cout << "| Privileges       | None              | CAP_IPC_LOCK      |\n";
    std::cout << "| Fragmentation    | May fail silently  | Pre-reserved      |\n";
    std::cout << "| Allocation speed | Slow (compaction) | Fast (reserved)   |\n";
    std::cout << "| Swap support     | Yes               | No                |\n";
    std::cout << "| Use case         | General purpose   | HPC, databases    |\n";
    std::cout << "+------------------+-------------------+-------------------+\n";

    // For C++ allocators:
    //   Boost.Interprocess provides huge page allocators
    //   jemalloc: MALLOC_CONF="thp:always" enables THP
    //   tcmalloc: set TCMALLOC_MEMFS_MALLOC_PATH=/dev/hugepages
}

```

---

## Notes

- TLB miss cost: ~7 cycles (L2 TLB hit) to ~100+ cycles (full page walk to DRAM).
- `perf stat -e dTLB-load-misses` measures TLB miss rate.
- For databases (large hash tables): huge pages can reduce query latency by 10-30%.
- Linux: check THP status with `cat /sys/kernel/mm/transparent_hugepage/enabled`.
- Windows: `VirtualAlloc` with `MEM_LARGE_PAGES` (requires `SeLockMemoryPrivilege`).
