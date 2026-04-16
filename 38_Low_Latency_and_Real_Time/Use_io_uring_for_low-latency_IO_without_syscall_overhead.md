# Use io_uring for low-latency IO without syscall overhead

**Category:** Low Latency and Real Time  
**Standard:** C++17  
**Reference:** <https://kernel.dk/io_uring.pdf>  

---

## Topic Overview

### The Problem with Traditional I/O

Every traditional I/O operation requires a **system call** — a context switch from user to kernel mode (~1-5 µs). For high-frequency trading or network servers processing millions of operations per second, this overhead is unacceptable:

```cpp

Traditional:  read() → [syscall] → kernel → copy → [return] → user
              ↑ context switch (~1 µs)              ↑ context switch

io_uring:     submit to SQ ring (user-space) → kernel polls → CQ ring (user-space)
              ↑ no syscall needed!

```

### io_uring Architecture

```cpp

User Space                           Kernel Space
┌──────────────┐                    ┌──────────────┐
│ Submission   │  shared memory     │              │
│ Queue (SQ)   │ ──────────────── → │  io_uring    │
│              │                    │  worker      │
│ Completion   │  shared memory     │  threads     │
│ Queue (CQ)   │ ← ──────────────  │              │
└──────────────┘                    └──────────────┘

SQ and CQ are ring buffers in shared memory.
After setup, submissions and completions need ZERO syscalls.

```

### Basic io_uring Setup with liburing

```cpp

#include <liburing.h>
#include <fcntl.h>
#include <cstring>
#include <stdexcept>

class IoUring {
    struct io_uring ring_;

public:
    explicit IoUring(unsigned queue_depth = 256, unsigned flags = 0) {
        int ret = io_uring_queue_init(queue_depth, &ring_, flags);
        if (ret < 0) {
            throw std::runtime_error("io_uring_queue_init failed: " +
                                     std::string(strerror(-ret)));
        }
    }

    ~IoUring() {
        io_uring_queue_exit(&ring_);
    }

    IoUring(const IoUring&) = delete;
    IoUring& operator=(const IoUring&) = delete;

    // Submit a read operation (non-blocking)
    void submit_read(int fd, void* buf, size_t len, off_t offset, uint64_t user_data) {
        struct io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        if (!sqe) {
            throw std::runtime_error("SQ ring full");
        }
        io_uring_prep_read(sqe, fd, buf, len, offset);
        io_uring_sqe_set_data64(sqe, user_data);
    }

    // Submit a write operation (non-blocking)
    void submit_write(int fd, const void* buf, size_t len, off_t offset, uint64_t user_data) {
        struct io_uring_sqe* sqe = io_uring_get_sqe(&ring_);
        if (!sqe) {
            throw std::runtime_error("SQ ring full");
        }
        io_uring_prep_write(sqe, fd, buf, len, offset);
        io_uring_sqe_set_data64(sqe, user_data);
    }

    // Flush submissions to kernel
    int submit() {
        return io_uring_submit(&ring_);
    }

    // Wait for a completion
    struct Completion {
        int result;       // bytes transferred or negative errno
        uint64_t user_data;
    };

    Completion wait_completion() {
        struct io_uring_cqe* cqe;
        int ret = io_uring_wait_cqe(&ring_, &cqe);
        if (ret < 0) {
            throw std::runtime_error("wait_cqe failed");
        }
        Completion comp{cqe->res, io_uring_cqe_get_data64(cqe)};
        io_uring_cqe_seen(&ring_, cqe);
        return comp;
    }

    // Poll for completions without blocking
    std::optional<Completion> try_completion() {
        struct io_uring_cqe* cqe;
        int ret = io_uring_peek_cqe(&ring_, &cqe);
        if (ret < 0) return std::nullopt;
        Completion comp{cqe->res, io_uring_cqe_get_data64(cqe)};
        io_uring_cqe_seen(&ring_, cqe);
        return comp;
    }
};

```

### Low-Latency Network Server with io_uring

```cpp

#include <liburing.h>
#include <netinet/in.h>
#include <sys/socket.h>

enum EventType : uint64_t { ACCEPT = 1, READ = 2, WRITE = 3 };

struct Connection {
    int fd;
    char buffer[4096];
    EventType pending_op;
};

void run_server(int listen_fd) {
    IoUring ring(1024, IORING_SETUP_SQPOLL);  // Kernel-side polling!
    // IORING_SETUP_SQPOLL: kernel thread polls the SQ ring
    // → zero syscalls for submission after setup

    // Submit initial accept
    struct io_uring_sqe* sqe = io_uring_get_sqe(&ring.raw());
    io_uring_prep_accept(sqe, listen_fd, nullptr, nullptr, 0);
    sqe->user_data = ACCEPT;
    io_uring_submit(&ring.raw());

    while (true) {
        struct io_uring_cqe* cqe;
        io_uring_wait_cqe(&ring.raw(), &cqe);

        switch (cqe->user_data & 0xFF) {
        case ACCEPT: {
            int client_fd = cqe->res;
            // Submit read on new connection
            auto* conn = new Connection{client_fd, {}, READ};
            sqe = io_uring_get_sqe(&ring.raw());
            io_uring_prep_recv(sqe, client_fd, conn->buffer, sizeof(conn->buffer), 0);
            sqe->user_data = reinterpret_cast<uint64_t>(conn);

            // Re-arm accept
            sqe = io_uring_get_sqe(&ring.raw());
            io_uring_prep_accept(sqe, listen_fd, nullptr, nullptr, 0);
            sqe->user_data = ACCEPT;
            break;
        }
        case READ: {
            auto* conn = reinterpret_cast<Connection*>(cqe->user_data);
            if (cqe->res <= 0) {
                close(conn->fd);
                delete conn;
                break;
            }
            // Echo back — submit write
            sqe = io_uring_get_sqe(&ring.raw());
            io_uring_prep_send(sqe, conn->fd, conn->buffer, cqe->res, 0);
            conn->pending_op = WRITE;
            sqe->user_data = reinterpret_cast<uint64_t>(conn);
            break;
        }
        }

        io_uring_cqe_seen(&ring.raw(), cqe);
        io_uring_submit(&ring.raw());  // With SQPOLL, this may be a no-op
    }
}

```

### io_uring vs epoll Performance

| Metric | epoll | io_uring | io_uring + SQPOLL |
| --- | --- | --- | --- |
| Syscalls per I/O | 2 (epoll_wait + read/write) | 1 (io_uring_enter) | 0 (kernel polls) |
| Latency (p99) | ~5 µs | ~2 µs | ~1 µs |
| Throughput | ~1M ops/s | ~2M ops/s | ~3M ops/s |
| CPU overhead | Moderate | Low | Lowest (dedicated kernel thread) |

### Registered Buffers and Files

```cpp

// Pre-register buffers — avoids per-I/O buffer mapping overhead
struct iovec iovecs[NUM_BUFFERS];
for (int i = 0; i < NUM_BUFFERS; i++) {
    iovecs[i].iov_base = aligned_alloc(4096, BUFFER_SIZE);
    iovecs[i].iov_len = BUFFER_SIZE;
}
io_uring_register_buffers(&ring, iovecs, NUM_BUFFERS);

// Use registered buffer index instead of pointer
struct io_uring_sqe* sqe = io_uring_get_sqe(&ring);
io_uring_prep_read_fixed(sqe, fd, iovecs[0].iov_base, BUFFER_SIZE, 0,
                          /*buf_index=*/0);

// Pre-register file descriptors
int fds[] = {fd1, fd2, fd3};
io_uring_register_files(&ring, fds, 3);
// Use IOSQE_FIXED_FILE flag with registered fd index

```

---

## Self-Assessment

### Q1: How does io_uring achieve zero-syscall I/O

io_uring uses **shared memory ring buffers** between user space and kernel space. The application writes submission queue entries (SQEs) directly into the shared SQ ring. With `IORING_SETUP_SQPOLL`, a kernel thread continuously polls the SQ ring for new entries and processes them without any system call from user space. Completions appear in the CQ ring, which the application reads from shared memory. After initial setup, the entire I/O path avoids context switches.

### Q2: When should you use `IORING_SETUP_SQPOLL`

Use SQPOLL when you need **absolute minimum latency** and can afford dedicating a CPU core to the kernel polling thread. It eliminates the `io_uring_enter` syscall for submissions. The tradeoff: it consumes 100% of one CPU core even when idle (the kernel thread spins). Best for: HFT, real-time networking, high-throughput storage. Avoid when: the system is CPU-constrained or I/O is infrequent.

### Q3: What are registered buffers and why do they improve performance

Without registration, every I/O operation requires the kernel to **map user-space buffer pages** into kernel address space (page pinning). With `io_uring_register_buffers`, the kernel pins the pages once during registration. Subsequent I/O operations using `io_uring_prep_read_fixed` reference the pre-registered buffer by index, skipping the per-I/O mapping overhead. This can save ~0.5-1 µs per operation.

---

## Notes

- io_uring is Linux-only (kernel 5.1+); on Windows, use IOCP; on macOS, use kqueue
- Always check `io_uring_queue_init_params` for feature support on the running kernel
- Use `io_uring_prep_multishot_accept` for accept without re-arming
- io_uring supports `IORING_OP_MSG_RING` for inter-ring communication (kernel 5.18+)
- Security: io_uring can be restricted with seccomp — some container runtimes disable it
