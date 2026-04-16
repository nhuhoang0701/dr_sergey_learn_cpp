# Use io_uring for async I/O with minimal syscall overhead

**Category:** Networking & I/O  
**Item:** #728  
**Reference:** <https://kernel.dk/io_uring.pdf>  

---

## Topic Overview

This topic focuses on batch submission and SQPOLL mode — complementing #640 (basic setup + CQ polling) and #551 (C++ RAII wrapper + SQ/CQ model).

```cpp

Batch submission (multiple ops, one syscall):
  sqe1 = get_sqe()  -> prep_read(file1)
  sqe2 = get_sqe()  -> prep_read(file2)
  sqe3 = get_sqe()  -> prep_write(file3)
  io_uring_submit()  -> ONE syscall submits all 3!

SQPOLL mode (zero submit syscalls):
  Kernel thread continuously polls the SQ ring
  User writes SQE -> kernel picks it up automatically
  No io_uring_enter() needed for submission!

```

| Mode | Submit cost | Complete cost | Best for |
| --- | --- | --- | --- |
| Normal | io_uring_enter per batch | io_uring_wait_cqe | General use |
| SQPOLL | Zero (kernel polls) | peek_cqe (no syscall) | Ultra-low latency |
| IOPOLL | io_uring_enter per batch | Busy-poll CQ | NVMe SSD |
| SQPOLL + IOPOLL | Zero | Busy-poll CQ | Highest throughput |

---

## Self-Assessment

### Q1: Batch submit multiple read operations

```cpp

// Requires: liburing
// Compile: g++ -std=c++20 -luring batch_submit.cpp
#include <iostream>
#include <vector>
#include <cstring>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    struct io_uring ring;
    io_uring_queue_init(64, &ring, 0);

    // Open multiple files
    const char* files[] = {"/etc/hostname", "/etc/os-release", "/etc/shells"};
    constexpr int NFILES = 3;
    int fds[NFILES];
    char bufs[NFILES][256] = {};

    for (int i = 0; i < NFILES; ++i) {
        fds[i] = open(files[i], O_RDONLY);
        if (fds[i] < 0) { perror(files[i]); continue; }

        // Prepare SQE for each file (no syscall yet!)
        auto* sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fds[i], bufs[i], 255, 0);
        sqe->user_data = i;  // tag to identify in completion
    }

    // Submit ALL requests in ONE syscall
    int submitted = io_uring_submit(&ring);
    std::cout << "Submitted " << submitted << " reads in 1 syscall\n";

    // Harvest completions
    for (int i = 0; i < submitted; ++i) {
        struct io_uring_cqe* cqe;
        io_uring_wait_cqe(&ring, &cqe);

        int idx = cqe->user_data;
        if (cqe->res >= 0) {
            bufs[idx][cqe->res] = '\0';
            std::cout << files[idx] << " (" << cqe->res << " bytes): "
                      << bufs[idx] << '\n';
        } else {
            std::cerr << files[idx] << ": error " << strerror(-cqe->res) << '\n';
        }
        io_uring_cqe_seen(&ring, cqe);
    }

    for (int i = 0; i < NFILES; ++i) close(fds[i]);
    io_uring_queue_exit(&ring);
}

```

### Q2: SQPOLL mode — zero syscall submission

```cpp

#include <iostream>
#include <cstring>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // SQPOLL: kernel thread continuously polls the submission ring
    struct io_uring ring;
    struct io_uring_params params{};
    params.flags = IORING_SETUP_SQPOLL;
    params.sq_thread_idle = 10000;  // kernel thread sleeps after 10s idle

    int ret = io_uring_queue_init_params(64, &ring, &params);
    if (ret < 0) {
        std::cerr << "SQPOLL requires CAP_SYS_NICE or root: "
                  << strerror(-ret) << '\n';
        // Fallback
        io_uring_queue_init(64, &ring, 0);
        std::cout << "Falling back to normal mode\n";
    } else {
        std::cout << "SQPOLL mode active!\n";
        std::cout << "  SQ thread CPU: " << params.sq_thread_cpu << '\n';
    }

    int fd = open("/etc/hostname", O_RDONLY);
    char buf[256] = {};

    auto* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, 255, 0);
    sqe->user_data = 1;

    // With SQPOLL: kernel thread picks up the SQE automatically
    // io_uring_submit() is optional (just wakes the thread if idle)
    io_uring_submit(&ring);  // wake SQ thread if sleeping

    // Poll CQ for completion (no syscall)
    struct io_uring_cqe* cqe;
    while (io_uring_peek_cqe(&ring, &cqe) != 0) {
        // busy-poll (in production: use io_uring_wait_cqe)
    }

    std::cout << "Read " << cqe->res << " bytes: " << buf;
    io_uring_cqe_seen(&ring, cqe);

    std::cout << "\nSQPOLL data path (zero syscalls):\n";
    std::cout << "  1. User writes SQE to shared ring memory\n";
    std::cout << "  2. Kernel SQ thread polls ring, submits I/O\n";
    std::cout << "  3. Kernel writes CQE to shared ring memory\n";
    std::cout << "  4. User reads CQE from shared memory (peek)\n";
    std::cout << "  Total syscalls: 0 (in steady-state)!\n";

    close(fd);
    io_uring_queue_exit(&ring);
}

```

### Q3: io_uring vs epoll+read throughput

**Benchmark comparison for random 4KB reads from NVMe SSD:**

| Metric | epoll + pread | io_uring (normal) | io_uring + SQPOLL |
| --- | --- | --- | --- |
| IOPS | ~120K | ~400K | ~500K |
| CPU/IOPS | High (2 syscalls/op) | Medium (batched) | Low (zero syscall) |
| p99 latency | ~15µs | ~5µs | ~3µs |
| Batch efficiency | None | Excellent | Excellent |

**Why io_uring is faster:**

1. **Batch submission**: N operations per syscall vs 1 syscall per operation (epoll + read).
2. **No context switches for completion**: CQE is in shared memory.
3. **True async file I/O**: epoll doesn't work with regular files (falls back to thread pool); io_uring handles files natively.
4. **Registered buffers**: `io_uring_register_buffers()` pins memory pages, avoiding per-I/O page-table walks.
5. **Linked operations**: chain dependent ops (read -> process -> write) without returning to user space.

```cpp

// Conceptual benchmark structure:
// for (int i = 0; i < N; ++i) {
//     // epoll: pread(fd, buf, 4096, random_offset); // 1 syscall per read
//     // io_uring: prep_read + batch submit every 32 ops // ~1 syscall per 32 reads
// }
// io_uring wins by ~3-4x on NVMe, less on slow disks (I/O bound, not CPU bound)

```

---

## Notes

- Complementary to #640 (basic setup + completion) and #551 (C++ RAII wrapper).
- SQPOLL requires `CAP_SYS_NICE` capability (or root) on kernels < 5.12.
- `io_uring_register_buffers()` pins pages for zero-copy DMA (huge perf win for NVMe).
- Linked SQEs: `sqe->flags |= IOSQE_IO_LINK` chains operations.
- io_uring is now used in production by: Seastar, Tokio (Rust), libev, and SPDK.
