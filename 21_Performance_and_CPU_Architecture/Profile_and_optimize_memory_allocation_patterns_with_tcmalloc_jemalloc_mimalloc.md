# Profile and optimize memory allocation patterns with tcmalloc, jemalloc, mimalloc

**Category:** Performance & CPU Architecture  
**Standard:** C++17 (allocator-agnostic)  
**Reference:** <https://google.github.io/tcmalloc/> <https://jemalloc.net/> <https://microsoft.github.io/mimalloc/>  

---

## Topic Overview

### Why the Default Allocator Isn't Always Enough

The default `malloc`/`new` implementation (glibc's ptmalloc2 on Linux, MSVC's allocator on Windows) is a general-purpose allocator. For high-performance applications, it can be a bottleneck due to:

- **Lock contention** in multi-threaded allocation/deallocation.
- **Fragmentation** causing poor cache utilization.
- **Large working set** from metadata overhead.

### Allocator Comparison

| Allocator | Developer | Best For | Key Feature |
| --- | --- | --- | --- |
| **tcmalloc** | Google | Multi-threaded servers | Per-thread caches, huge page support |
| **jemalloc** | Meta/FreeBSD | General high-perf | Arena-based, low fragmentation |
| **mimalloc** | Microsoft | Latency-sensitive | Segment-based, free list sharding |
| **ptmalloc2** | glibc | General purpose | Default on Linux |

### Drop-In Replacement

All three can replace the system allocator without code changes:

```bash

# tcmalloc
LD_PRELOAD=/usr/lib/libtcmalloc.so ./my_app

# jemalloc
LD_PRELOAD=/usr/lib/libjemalloc.so ./my_app

# mimalloc
LD_PRELOAD=/usr/lib/libmimalloc.so ./my_app

# On Windows with mimalloc: link against mimalloc-override.lib

```

### CMake Integration

```cmake

# tcmalloc via find_package or FetchContent
find_package(tcmalloc REQUIRED)
target_link_libraries(myapp PRIVATE tcmalloc::tcmalloc)

# jemalloc
find_package(PkgConfig REQUIRED)
pkg_check_modules(JEMALLOC REQUIRED jemalloc)
target_link_libraries(myapp PRIVATE ${JEMALLOC_LIBRARIES})

# mimalloc
find_package(mimalloc REQUIRED)
target_link_libraries(myapp PRIVATE mimalloc)

```

### Profiling Allocation Patterns

**With jemalloc's built-in profiler:**

```bash

# Enable heap profiling
MALLOC_CONF="prof:true,prof_prefix:jeprof" ./my_app

# Analyze with jeprof
jeprof --svg ./my_app jeprof.*.heap > heap_profile.svg

```

**With tcmalloc's heap profiler:**

```cpp

#include <gperftools/heap-profiler.h>

int main() {
    HeapProfilerStart("my_profile");

    // ... application code ...
    // Periodic dumps:
    HeapProfilerDump("checkpoint1");

    HeapProfilerStop();
}

```

```bash

# Convert to visualization
pprof --svg ./my_app my_profile.0001.heap > heap.svg

```

**With `perf` for allocation hotspots:**

```bash

perf record -g -e probe:malloc ./my_app
perf report --sort=symbol,dso

```

### tcmalloc: Per-Thread Caching

tcmalloc maintains per-thread freelists for small allocations, eliminating lock contention:

```cpp

Thread 1: [local cache] → [central freelist] → [page heap]
Thread 2: [local cache] → [central freelist] → [page heap]
Thread 3: [local cache] → [central freelist] → [page heap]

```

Small allocations (<256KB) are served entirely from the thread-local cache — **zero locking**.

### jemalloc: Arena-Based Allocation

jemalloc uses multiple arenas with size-class bins:

```cpp

Arena 0: [bins: 8B, 16B, 32B, ... 14KB] [large: 14KB-4MB] [huge: >4MB]
Arena 1: [bins: 8B, 16B, 32B, ... 14KB] [large: 14KB-4MB] [huge: >4MB]
...

```

Threads are distributed across arenas, reducing contention.

### mimalloc: Free List Sharding

mimalloc shards free lists per page per size class, achieving excellent cache locality:

```cpp

// mimalloc-specific APIs (optional — also works as drop-in)
#include <mimalloc.h>

void* p = mi_malloc(1024);
mi_free(p);

// Aligned allocation
void* aligned = mi_malloc_aligned(4096, 64);
mi_free(aligned);

```

---

## Self-Assessment

### Q1: How do you benchmark different allocators

```bash

# Run the same workload with different allocators and compare:
time LD_PRELOAD="" ./my_app                          # default
time LD_PRELOAD=/usr/lib/libtcmalloc.so ./my_app     # tcmalloc
time LD_PRELOAD=/usr/lib/libjemalloc.so ./my_app      # jemalloc
time LD_PRELOAD=/usr/lib/libmimalloc.so ./my_app      # mimalloc

# Also measure:
# - RSS (resident set size): ps -o rss
# - Allocation rate: perf stat -e malloc calls
# - Lock contention: perf stat -e context-switches

```

### Q2: When should you prefer jemalloc over tcmalloc

- **jemalloc**: lower fragmentation for long-running services with diverse allocation sizes (databases, web servers).
- **tcmalloc**: better for workloads with many small same-size allocations from many threads (RPC servers, microservices).
- **mimalloc**: best latency characteristics for interactive/real-time applications.

### Q3: How can you profile per-allocation-site memory usage

With jemalloc:

```bash

MALLOC_CONF="prof:true,lg_prof_interval:30,prof_prefix:heap" ./my_app
jeprof --text ./my_app heap.*.heap

```

With tcmalloc:

```bash

HEAPPROFILE=/tmp/hp ./my_app
pprof --text --alloc_space ./my_app /tmp/hp.0001.heap

```

Both show allocation call stacks with byte counts, allowing you to identify the hottest allocation sites.

---

## Notes

- Always benchmark with **your** workload — allocator performance varies dramatically by access pattern.
- `LD_PRELOAD` is the easiest way to test — no recompilation needed.
- For C++ specifically, `operator new`/`delete` go through `malloc`/`free`, so replacing `malloc` replaces all standard allocations.
- Consider combining a global allocator (tcmalloc/jemalloc) with local arena allocators (`std::pmr`) for hot paths.
