# Use memory-mapped files (mmap) for fast file I/O

**Category:** Networking & I/O  
**Item:** #642  
**Reference:** <https://man7.org/linux/man-pages/man2/mmap.2.html>  

---

## Topic Overview

Memory-mapped files map a file directly into the process's virtual address space. Instead of read()/write() syscalls, you access file data as a pointer — the kernel handles page faults and caching transparently.

```cpp

mmap data flow:
  Virtual address space     Physical RAM        Disk
  +------------------+     +--------+       +--------+
  | mapped region    | --> | page   | <---> | file   |
  | ptr[0..filesize] |     | cache  |       | blocks |
  +------------------+     +--------+       +--------+
  
  Access ptr[100]:      

    1. CPU generates virtual address
    2. If page not in RAM: page fault -> kernel loads from disk
    3. If page in RAM: direct access (no syscall!)

```

| Method | Syscalls per access | Copies | Random access |
| --- | --- | --- | --- |
| read() | 1 per call | 1 (kernel->user) | Seek + read |
| mmap | 0 (after initial fault) | 0 (shared pages) | Pointer arithmetic |
| pread() | 1 per call | 1 | Offset parameter |

---

## Self-Assessment

### Q1: Map a file read-only with mmap and access as byte array

```cpp

// Linux/POSIX example
#include <iostream>
#include <cstring>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // Open file
    int fd = open("/etc/hostname", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    // Get file size
    struct stat st;
    fstat(fd, &st);
    size_t file_size = st.st_size;

    // Map the entire file into memory
    void* addr = mmap(
        nullptr,          // let kernel choose address
        file_size,        // length to map
        PROT_READ,        // read-only access
        MAP_SHARED,       // shared with other processes (or MAP_PRIVATE)
        fd,               // file descriptor
        0                 // offset in file
    );

    if (addr == MAP_FAILED) {
        perror("mmap");
        close(fd);
        return 1;
    }

    // fd can be closed after mmap (mapping stays valid)
    close(fd);

    // Access file data as a byte array!
    const char* data = static_cast<const char*>(addr);
    std::cout << "File content (" << file_size << " bytes): ";
    std::cout.write(data, file_size);

    // Random access: just use pointer arithmetic
    if (file_size > 0)
        std::cout << "First byte: '" << data[0] << "'\n";
    if (file_size > 5)
        std::cout << "Byte 5: '" << data[5] << "'\n";

    // MUST unmap when done
    munmap(addr, file_size);
}
// No read() syscall needed!
// Page faults load data on demand from disk cache

```

### Q2: madvise hints for sequential access

```cpp

#include <iostream>
#include <cstring>
#include <chrono>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    const char* filepath = "/tmp/mmap_test.bin";

    // Create a test file (10 MB)
    {
        int fd = open(filepath, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        char buf[4096];
        memset(buf, 'A', sizeof(buf));
        for (int i = 0; i < 2560; ++i)  // 10 MB
            write(fd, buf, sizeof(buf));
        close(fd);
    }

    int fd = open(filepath, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    auto* addr = static_cast<const char*>(
        mmap(nullptr, st.st_size, PROT_READ, MAP_SHARED, fd, 0));
    close(fd);

    // madvise tells the kernel our access pattern
    // MADV_SEQUENTIAL: read-ahead aggressively, free pages behind us
    madvise(const_cast<char*>(addr), st.st_size, MADV_SEQUENTIAL);

    // Sequential scan
    auto t0 = std::chrono::high_resolution_clock::now();
    volatile long long sum = 0;
    for (off_t i = 0; i < st.st_size; ++i)
        sum += addr[i];  // sequential access -> kernel prefetches ahead
    auto t1 = std::chrono::high_resolution_clock::now();

    auto ms = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
    std::cout << "Sequential scan: " << ms.count() << " us ("
              << st.st_size / 1024 / 1024 << " MB)\n";

    // Other madvise hints:
    // MADV_RANDOM:     no read-ahead (for random access patterns)
    // MADV_WILLNEED:   prefetch pages (like explicit prefetch)
    // MADV_DONTNEED:   free pages (reduce RSS)
    // MADV_HUGEPAGE:   use transparent huge pages (2MB)
    // MADV_POPULATE_READ: fault all pages now (avoid later page faults)

    munmap(const_cast<char*>(addr), st.st_size);
    unlink(filepath);
}

```

### Q3: mmap vs read() throughput comparison

```cpp

#include <iostream>
#include <chrono>
#include <cstring>
#include <vector>
#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>

long long bench_read(const char* path) {
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    auto t0 = std::chrono::high_resolution_clock::now();
    std::vector<char> buf(4096);
    volatile long long sum = 0;
    ssize_t n;
    while ((n = read(fd, buf.data(), buf.size())) > 0) {
        for (ssize_t i = 0; i < n; ++i) sum += buf[i];
    }
    auto t1 = std::chrono::high_resolution_clock::now();
    close(fd);
    return std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
}

long long bench_mmap(const char* path) {
    int fd = open(path, O_RDONLY);
    struct stat st;
    fstat(fd, &st);

    auto* addr = static_cast<const char*>(
        mmap(nullptr, st.st_size, PROT_READ, MAP_SHARED, fd, 0));
    close(fd);
    madvise(const_cast<char*>(addr), st.st_size, MADV_SEQUENTIAL);

    auto t0 = std::chrono::high_resolution_clock::now();
    volatile long long sum = 0;
    for (off_t i = 0; i < st.st_size; ++i) sum += addr[i];
    auto t1 = std::chrono::high_resolution_clock::now();

    munmap(const_cast<char*>(addr), st.st_size);
    return std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
}

int main() {
    const char* path = "/tmp/mmap_bench.bin";
    // Create 100MB test file
    {
        int fd = open(path, O_CREAT | O_WRONLY | O_TRUNC, 0644);
        char buf[4096];
        memset(buf, 'X', sizeof(buf));
        for (int i = 0; i < 25600; ++i) write(fd, buf, sizeof(buf));
        close(fd);
    }

    auto t_read = bench_read(path);
    auto t_mmap = bench_mmap(path);

    std::cout << "100 MB sequential scan:\n";
    std::cout << "  read(): " << t_read / 1000 << " ms\n";
    std::cout << "  mmap(): " << t_mmap / 1000 << " ms\n";
    // Typical: mmap is 10-30% faster for sequential scans
    // (fewer syscalls, no user-space buffer copy)

    std::cout << "\nWhen mmap wins:\n";
    std::cout << "  - Random access to large files (>RAM: only pages in use loaded)\n";
    std::cout << "  - Repeated access (page cache shared with other processes)\n";
    std::cout << "  - Shared memory between processes (MAP_SHARED)\n";
    std::cout << "\nWhen read() wins:\n";
    std::cout << "  - Sequential one-pass (similar perf, simpler code)\n";
    std::cout << "  - Many small files (mmap setup overhead per file)\n";
    std::cout << "  - Need to control I/O timing (mmap faults are unpredictable)\n";

    unlink(path);
}

```

---

## Notes

- Complementary to #552 and #731 (other mmap/MapViewOfFile topics).
- `MAP_SHARED` shares modifications with other processes; `MAP_PRIVATE` uses copy-on-write.
- Windows equivalent: `CreateFileMapping()` + `MapViewOfFile()`.
- SIGBUS occurs if the file is truncated while mapped — handle this signal.
- For concurrent access: mmap is inherently thread-safe for reads; writes need synchronization.
