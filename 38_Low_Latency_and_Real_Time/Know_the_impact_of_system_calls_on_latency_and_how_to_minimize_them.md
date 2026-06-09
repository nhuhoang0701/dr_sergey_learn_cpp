# Know the impact of system calls on latency and how to minimize them

**Category:** Low Latency and Real Time  
**Standard:** C++17  
**Reference:** <https://man7.org/linux/man-pages/man2/syscalls.2.html>  

---

## Topic Overview

### System Call Cost Breakdown

A system call is not just a function call - it involves privilege level transition. The CPU has to switch from user mode (ring 3) to kernel mode (ring 0), which means saving your registers, switching the page table, executing the kernel handler, and then reversing all of that on the way back out. This is not optional overhead - the hardware enforces it.

```cpp
User Space                                    Kernel Space
┌──────────────┐  SYSCALL instruction  ┌──────────────────┐
│ Application  │ ─── (~100-200 ns) ──> │                  │
│              │                       │ Save registers    │
│              │                       │ Switch page table │
│              │                       │ Execute handler   │
│              │                       │ Restore state     │
│              │ <── (~100-200 ns) ─── │                  │
└──────────────┘  SYSRET instruction   └──────────────────┘

Minimum syscall overhead: ~200-400 ns (x86-64, Linux)
With Spectre mitigations: ~800-1500 ns (!!)
```

The Spectre mitigation numbers deserve attention. Post-2018, most production Linux systems run with KPTI and retpolines enabled, which more than doubles or triples the cost of every syscall. Code that was marginal before may be completely unusable after enabling mitigations. This makes syscall elimination even more valuable now than it was five years ago.

### Measuring System Call Costs

Before optimizing, measure. Here's a simple benchmark to see what syscalls actually cost on your machine with your current kernel configuration.

```cpp
#include <chrono>
#include <unistd.h>
#include <sys/syscall.h>

void measure_syscall_overhead() {
    constexpr int ITERATIONS = 1'000'000;

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < ITERATIONS; ++i) {
        // getpid() is one of the fastest syscalls (vDSO-cached)
        syscall(SYS_getpid);
    }
    auto end = std::chrono::high_resolution_clock::now();

    auto ns = std::chrono::duration_cast<std::chrono::nanoseconds>(end - start).count();
    printf("Average syscall: %ld ns\n", ns / ITERATIONS);
    // Typical result: 200-800 ns depending on CPU and mitigations
}
```

Note that `getpid()` is one of the fastest possible syscalls because it's typically implemented via the vDSO (see below). This gives you a lower bound. Real syscalls like `read()` and `write()` will be slower.

### Common Syscalls and Their Latency

| Syscall | Typical latency | Notes |
| --- | --- | --- |
| `getpid()` | ~0 ns (vDSO) | Cached in user space |
| `clock_gettime()` | ~20-50 ns (vDSO) | No kernel transition |
| `gettimeofday()` | ~20-50 ns (vDSO) | Same mechanism |
| `read()` (cache hit) | ~200-1000 ns | Page cache hit |
| `write()` (buffered) | ~200-500 ns | Copied to page cache |
| `mmap()` | ~1-10 µs | Page table manipulation |
| `futex()` (uncontended) | ~200-400 ns | Fast path |
| `epoll_wait()` | ~1-5 µs | Event polling |
| `open()` | ~5-50 µs | Path resolution, dentries |

### Technique 1: Use vDSO (Virtual Dynamic Shared Object)

The kernel maps a small library into every process that handles certain syscalls without crossing the kernel boundary. For these calls, there's no ring transition at all - the kernel data is accessible from user space via a shared memory region that the kernel updates atomically.

```cpp
#include <time.h>

void fast_timestamp() {
    struct timespec ts;
    // This does NOT enter the kernel — it's handled by vDSO
    clock_gettime(CLOCK_MONOTONIC, &ts);
    // ~20 ns on modern Linux
}

// vDSO-accelerated syscalls (Linux):
// - clock_gettime()
// - gettimeofday()
// - getcpu()
// - time()
```

The practical implication is that you should use `clock_gettime(CLOCK_MONOTONIC, ...)` for timestamps, not `gettimeofday()` directly, and definitely not `time()`. They all go through vDSO, but `CLOCK_MONOTONIC` is the right clock for latency measurements (it can't jump backward like `CLOCK_REALTIME` can during NTP adjustments).

### Technique 2: Batch System Calls

If you must call into the kernel, do it as infrequently as possible by batching operations. The `writev` scatter-gather syscall lets you write multiple non-contiguous memory regions in a single kernel entry.

```cpp
#include <sys/uio.h>

// BAD: 4 syscalls
void write_four_fields_bad(int fd) {
    write(fd, header, header_len);   // syscall 1
    write(fd, field1, field1_len);   // syscall 2
    write(fd, field2, field2_len);   // syscall 3
    write(fd, trailer, trailer_len); // syscall 4
}

// GOOD: 1 syscall with scatter-gather I/O
void write_four_fields_good(int fd) {
    struct iovec iov[4] = {
        {header,  header_len},
        {field1,  field1_len},
        {field2,  field2_len},
        {trailer, trailer_len},
    };
    writev(fd, iov, 4);  // Single syscall!
}
```

With Spectre mitigations, replacing 4 syscalls with 1 can save 2-4 µs per operation. In a system processing 100,000 operations per second, that's 200-400 ms of saved latency per second.

### Technique 3: Memory-Mapped I/O

After the initial `mmap` setup call, reading and writing a memory-mapped file is indistinguishable from normal memory access - no syscall required. The kernel handles the caching and flushing transparently.

```cpp
#include <sys/mman.h>
#include <fcntl.h>

class MappedFile {
    void* addr_ = MAP_FAILED;
    size_t size_;

public:
    MappedFile(const char* path, size_t size) : size_(size) {
        int fd = open(path, O_RDWR | O_CREAT, 0644);
        ftruncate(fd, size);
        addr_ = mmap(nullptr, size, PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0);
        close(fd);  // fd can be closed after mmap

        // Prefault pages to avoid page faults during hot path
        madvise(addr_, size, MADV_WILLNEED);
        // Or: MAP_POPULATE flag in mmap() to fault all pages upfront
    }

    ~MappedFile() {
        if (addr_ != MAP_FAILED) munmap(addr_, size_);
    }

    // Access file as memory — no read()/write() syscalls!
    char* data() { return static_cast<char*>(addr_); }

    // Force write-back to disk
    void sync() { msync(addr_, size_, MS_SYNC); }
};

// Usage: read/write file without any syscalls after setup
void hot_path(MappedFile& file) {
    // Direct memory access — zero syscalls
    auto* record = reinterpret_cast<Record*>(file.data() + offset);
    record->timestamp = get_timestamp();
    record->value = new_value;
    // Data is written to the page cache; kernel flushes asynchronously
}
```

The `MADV_WILLNEED` hint asks the kernel to prefault the pages in the background so they're already in the TLB and page cache when you need them. Without it, the first access to each 4KB page triggers a page fault - which is a kernel transition, defeating the whole purpose.

### Technique 4: Pre-allocate Everything

Dynamic memory allocation (`malloc`, `new`) can call `mmap` or `brk` syscalls when the heap needs to grow. Pre-allocate your working memory during startup so the hot path never touches the allocator.

```cpp
// BAD: malloc may call mmap or brk (syscalls)
void hot_path_bad() {
    auto* data = new char[4096];  // Possible syscall!
    process(data);
    delete[] data;
}

// GOOD: Pre-allocate a memory pool
class MemoryPool {
    alignas(64) char pool_[1024 * 1024];  // 1 MB pre-allocated
    size_t offset_ = 0;

public:
    void* allocate(size_t size) {
        size = (size + 63) & ~63;  // Align to cache line
        if (offset_ + size > sizeof(pool_)) return nullptr;
        void* ptr = pool_ + offset_;
        offset_ += size;
        return ptr;
    }

    void reset() { offset_ = 0; }  // "Free" everything at once
};

// Also: use mlockall(MCL_CURRENT | MCL_FUTURE) to prevent page faults
```

`mlockall(MCL_CURRENT | MCL_FUTURE)` is important: it tells the kernel to keep all of your process's pages in physical memory and never swap them to disk. Without it, pages that haven't been accessed recently can be swapped out, and accessing them on the hot path triggers a page fault - which is a kernel transition that can cost microseconds.

### Technique 5: Avoid Logging/Allocation in Hot Path

Logging is one of the most common sources of accidental syscalls in hot paths. `printf` calls `write()`, which is a syscall. The solution is to log to a ring buffer in memory and drain that buffer from a dedicated background thread.

```cpp
// BAD: printf calls write() (syscall) and may allocate
void on_market_data_bad(const Quote& q) {
    printf("Quote: %s %.2f\n", q.symbol, q.price);  // SYSCALL!
    process_quote(q);
}

// GOOD: Log to a ring buffer, drain from a separate thread
class AsyncLogger {
    struct Entry { uint64_t timestamp; char msg[120]; };
    SPSCQueue<Entry, 65536> queue_;  // Lock-free, no syscalls

public:
    void log(const char* msg) {
        Entry e;
        e.timestamp = __rdtsc();
        std::strncpy(e.msg, msg, sizeof(e.msg) - 1);
        queue_.push(e);  // No syscall — just a memory write
    }

    // Called from a background thread
    void drain(int fd) {
        Entry e;
        while (queue_.pop(e)) {
            dprintf(fd, "[%lu] %s\n", e.timestamp, e.msg);
        }
    }
};
```

The hot path's `log()` call is now just a few memory writes - no kernel involvement whatsoever. The background thread pays the syscall cost but that's fine because it's not on the latency-critical path.

---

## Self-Assessment

### Q1: Why are Spectre mitigations particularly painful for syscall-heavy workloads

Spectre mitigations (KPTI/KAISER, retpolines, IBRS) add overhead to every kernel entry/exit:

- **KPTI** flushes the TLB on user<->kernel transitions (~200-400 ns added)
- **Retpolines** prevent speculative execution through indirect branches
- **IBRS/STIBP** restrict branch prediction across privilege boundaries

A syscall that cost ~200 ns pre-Spectre may now cost ~800-1500 ns. This makes syscall elimination even more important for low-latency applications. Reducing from 10 syscalls to 1 on a mitigated system saves ~10 µs.

### Q2: What is vDSO and which syscalls benefit from it

vDSO (Virtual Dynamic Shared Object) is a small shared library that the kernel maps into every process's address space. Certain "read-only" syscalls are implemented entirely in user space via vDSO, avoiding the kernel transition. On Linux: `clock_gettime`, `gettimeofday`, `getcpu`, and `time`. These complete in ~20-50 ns instead of ~200-800 ns. The kernel updates the vDSO data via shared memory.

### Q3: How do you achieve zero-syscall I/O

Three approaches: (1) **Memory-mapped files** (`mmap`) - after initial setup, reads and writes are plain memory accesses. (2) **io_uring with SQPOLL** - kernel thread polls for I/O submissions without any user-space syscall. (3) **Shared memory IPC** - communication between processes via mapped shared memory regions, no read/write syscalls. All three require upfront setup syscalls but eliminate per-operation syscalls in the hot path.

---

## Notes

- Use `strace -c ./my_app` to count syscalls and their latency per type
- `perf stat -e syscalls:sys_enter_* ./my_app` shows syscall distribution
- `mlockall(MCL_CURRENT | MCL_FUTURE)` prevents page faults (which are implicit syscalls)
- Hugepages reduce TLB misses (which cause page-walk syscalls): `madvise(addr, len, MADV_HUGEPAGE)`
- On Windows, similar principles apply - avoid `ReadFile`/`WriteFile` in hot paths; use memory-mapped files
