# Use io_uring for asynchronous I/O without syscall overhead

**Category:** Networking & I/O  
**Item:** #640  
**Reference:** <https://kernel.dk/io_uring.pdf>  

---

## Topic Overview

io_uring is Linux's newest async I/O interface (kernel 5.1+). It uses shared memory ring buffers between user space and kernel, allowing I/O submission and completion without syscalls in the hot path.

```cpp

io_uring architecture:
  User space          Kernel
  +--------+        +--------+
  |  SQE   | -----> |  SQ    |  Submission Queue
  | (write) |        | (read) |
  +--------+        +--------+
                         |
                    [kernel does I/O]
                         |
  +--------+        +--------+
  |  CQE   | <----- |  CQ    |  Completion Queue
  | (read) |        | (write) |
  +--------+        +--------+

  Submit: write SQE to shared ring -> io_uring_enter() or SQPOLL
  Complete: read CQE from shared ring (NO syscall needed!)

```

| I/O model | Submit syscall | Complete syscall | Batching |
| --- | --- | --- | --- |
| Blocking read/write | 1 per op | N/A | No |
| epoll + non-blocking | epoll_wait + read | N/A | Per-event |
| io_uring | io_uring_enter (batched) | None (CQ polling) | Yes |
| io_uring + SQPOLL | None (kernel polls SQ) | None (CQ polling) | Yes |

---

## Self-Assessment

### Q1: Set up io_uring and submit a read request

```cpp

// Requires: liburing (apt install liburing-dev)
// Compile: g++ -std=c++20 -luring uring_read.cpp
#include <iostream>
#include <cstring>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>

int main() {
    // 1. Initialize io_uring with 32 SQ entries
    struct io_uring ring;
    int ret = io_uring_queue_init(32, &ring, 0);
    if (ret < 0) {
        std::cerr << "io_uring_queue_init: " << strerror(-ret) << '\n';
        return 1;
    }

    // 2. Open a file
    int fd = open("/etc/hostname", O_RDONLY);
    if (fd < 0) { perror("open"); return 1; }

    // 3. Prepare a read SQE (Submission Queue Entry)
    char buf[256] = {};
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
    io_uring_prep_read(sqe, fd, buf, sizeof(buf) - 1, 0);  // read at offset 0
    sqe->user_data = 42;  // tag for identifying in CQE

    // 4. Submit the request (single syscall for all pending SQEs)
    ret = io_uring_submit(&ring);  // io_uring_enter() under the hood
    std::cout << "Submitted " << ret << " request(s)\n";

    // 5. Wait for completion (read from CQ ring)
    struct io_uring_cqe* cqe;
    ret = io_uring_wait_cqe(&ring, &cqe);  // blocks until CQE available
    if (ret < 0) {
        std::cerr << "wait_cqe: " << strerror(-ret) << '\n';
    } else {
        std::cout << "Completed: user_data=" << cqe->user_data
                  << " result=" << cqe->res << " bytes\n";
        std::cout << "Data: " << buf;
        io_uring_cqe_seen(&ring, cqe);  // mark CQE as consumed
    }

    close(fd);
    io_uring_queue_exit(&ring);
}
// Output:
//   Submitted 1 request(s)
//   Completed: user_data=42 result=12 bytes
//   Data: my-hostname

```

### Q2: Harvest completions without syscall (CQ polling)

```cpp

#include <iostream>
#include <liburing.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>

int main() {
    // Setup with SQPOLL (kernel thread polls submission queue)
    struct io_uring ring;
    struct io_uring_params params{};
    params.flags = IORING_SETUP_SQPOLL;  // kernel polls SQ automatically
    params.sq_thread_idle = 2000;         // SQ thread sleeps after 2s idle

    int ret = io_uring_queue_init_params(256, &ring, &params);
    if (ret < 0) {
        // SQPOLL requires root or CAP_SYS_NICE
        std::cerr << "init with SQPOLL: " << strerror(-ret)
                  << " (needs root)\n";
        // Fallback to normal mode
        io_uring_queue_init(256, &ring, 0);
    }

    // Submit multiple reads
    int fd = open("/etc/hostname", O_RDONLY);
    char bufs[4][64] = {};

    for (int i = 0; i < 4; ++i) {
        auto* sqe = io_uring_get_sqe(&ring);
        io_uring_prep_read(sqe, fd, bufs[i], 63, 0);
        sqe->user_data = i;
    }
    io_uring_submit(&ring);

    // Harvest completions by polling CQ (no syscall!)
    // The CQ is in shared memory; we just read from it.
    int completed = 0;
    while (completed < 4) {
        struct io_uring_cqe* cqe;

        // io_uring_peek_cqe: non-blocking check (NO syscall)
        ret = io_uring_peek_cqe(&ring, &cqe);
        if (ret == 0) {
            std::cout << "CQE[" << cqe->user_data << "]: "
                      << cqe->res << " bytes\n";
            io_uring_cqe_seen(&ring, cqe);
            ++completed;
        }
        // In real code: io_uring_wait_cqe_nr for batch waiting
    }

    close(fd);
    io_uring_queue_exit(&ring);

    std::cout << "\nZero-syscall hot path:\n";
    std::cout << "  Submit: write SQE to shared memory ring\n";
    std::cout << "  Complete: read CQE from shared memory ring\n";
    std::cout << "  With SQPOLL: kernel thread polls SQ -> no submit syscall\n";
    std::cout << "  With CQ polling: peek_cqe reads shared mem -> no wait syscall\n";
}

```

### Q3: io_uring vs epoll throughput comparison

| Metric | epoll + read() | io_uring | io_uring + SQPOLL |
| --- | --- | --- | --- |
| Syscalls per I/O | 2 (epoll_wait + read) | 1 (io_uring_enter, batched) | 0 (kernel polls) |
| Batch submission | No | Yes (multiple SQEs) | Yes |
| File I/O | Must use thread pool | Native async | Native async |
| Network I/O | Excellent | Good (improving) | Good |
| IOPS (4K random reads) | ~100K | ~400K | ~500K |
| Latency (p99) | ~10µs | ~3µs | ~2µs |
| CPU usage at 100K IOPS | ~30% | ~15% | ~20% (SQ thread) |
| Kernel version | 2.6+ | 5.1+ | 5.11+ |

**When to use io_uring over epoll:**

- High-IOPS file I/O (databases, storage engines)
- When syscall overhead matters (>100K ops/sec)
- Mixed file + network I/O in one event loop
- When batching submissions (N ops per enter())

**When epoll is still fine:**

- Network-only servers (<50K connections)
- When portability matters (io_uring is Linux-only)
- Simple request/response protocols
- When using Asio (abstracts the backend)

---

## Notes

- Complementary to #728 (SQPOLL mode) and #551 (C++ RAII wrapper).
- io_uring supports: read, write, accept, connect, send, recv, close, timeout, and more.
- liburing is the recommended C wrapper (header-only, easy to wrap in C++ RAII).
- Security: SQPOLL mode requires elevated privileges (CAP_SYS_NICE or root).
- Asio 2.0 has experimental io_uring backend (`ASIO_HAS_IO_URING`).
