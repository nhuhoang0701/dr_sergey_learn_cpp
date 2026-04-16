# Design inter-process communication (IPC) architecture in C++

**Category:** Project Architecture

---

## Topic Overview

**IPC** enables separate processes to exchange data and coordinate. C++ applications use IPC for multi-process architectures (e.g., Chrome's process-per-tab), daemon/client communication, and distributed systems. The choice of IPC mechanism depends on latency requirements, data volume, and platform constraints.

### IPC Mechanisms Comparison

| Mechanism | Latency | Throughput | Platforms | Complexity |
| --- | --- | --- | --- | --- |
| **Shared Memory** | ~100ns | Highest (memcpy) | POSIX, Win32 | High (sync needed) |
| **Unix Domain Socket** | ~1µs | Very high | POSIX | Medium |
| **Named Pipe (FIFO)** | ~1µs | High | All | Low |
| **TCP Socket** | ~10µs+ | Network-bound | All | Medium |
| **Message Queues** | ~1µs | High | POSIX | Medium |
| **gRPC** | ~100µs | Network-bound | All | High (generated) |
| **Memory-mapped file** | ~100ns | Highest | All | Medium |

---

## Self-Assessment

### Q1: Implement shared memory IPC with synchronization

**Answer:**

```cpp

#include <sys/mman.h>
#include <sys/stat.h>
#include <fcntl.h>
#include <unistd.h>
#include <cstring>
#include <atomic>
#include <iostream>

// === Shared memory ring buffer for IPC ===
struct SharedBuffer {
    static constexpr size_t CAPACITY = 4096;

    // Lock-free SPSC (single producer, single consumer)
    alignas(64) std::atomic<size_t> write_pos{0};
    alignas(64) std::atomic<size_t> read_pos{0};
    char data[CAPACITY];

    bool write(const void* src, size_t len) {
        size_t w = write_pos.load(std::memory_order_relaxed);
        size_t r = read_pos.load(std::memory_order_acquire);
        size_t available = CAPACITY - (w - r);
        if (len + sizeof(size_t) > available) return false;  // Full

        size_t offset = w % CAPACITY;
        // Write length prefix + data
        std::memcpy(data + offset, &len, sizeof(size_t));
        std::memcpy(data + offset + sizeof(size_t), src, len);
        write_pos.store(w + sizeof(size_t) + len,
                        std::memory_order_release);
        return true;
    }

    bool read(void* dst, size_t& len) {
        size_t w = write_pos.load(std::memory_order_acquire);
        size_t r = read_pos.load(std::memory_order_relaxed);
        if (w == r) return false;  // Empty

        size_t offset = r % CAPACITY;
        std::memcpy(&len, data + offset, sizeof(size_t));
        std::memcpy(dst, data + offset + sizeof(size_t), len);
        read_pos.store(r + sizeof(size_t) + len,
                       std::memory_order_release);
        return true;
    }
};

// === Producer process ===
void producer() {
    int fd = shm_open("/my_ipc", O_CREAT | O_RDWR, 0666);
    ftruncate(fd, sizeof(SharedBuffer));
    auto* buf = static_cast<SharedBuffer*>(
        mmap(nullptr, sizeof(SharedBuffer),
             PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0));
    new (buf) SharedBuffer{};  // Placement new

    std::string msg = "Hello from producer";
    buf->write(msg.data(), msg.size());
    // ...
    munmap(buf, sizeof(SharedBuffer));
    close(fd);
}

// === Consumer process ===
void consumer() {
    int fd = shm_open("/my_ipc", O_RDWR, 0666);
    auto* buf = static_cast<SharedBuffer*>(
        mmap(nullptr, sizeof(SharedBuffer),
             PROT_READ | PROT_WRITE, MAP_SHARED, fd, 0));

    char data[256];
    size_t len;
    if (buf->read(data, len)) {
        std::cout << std::string(data, len) << "\n";
    }
    munmap(buf, sizeof(SharedBuffer));
    close(fd);
    shm_unlink("/my_ipc");
}

```

### Q2: Implement Unix domain socket IPC with message framing

**Answer:**

```cpp

#include <sys/socket.h>
#include <sys/un.h>
#include <unistd.h>
#include <cstring>
#include <string>

// === Message format: [4-byte length][payload] ===
class SocketIPC {
public:
    static bool send_message(int fd, const std::string& msg) {
        uint32_t len = static_cast<uint32_t>(msg.size());
        // Send length prefix
        if (::send(fd, &len, sizeof(len), 0) != sizeof(len))
            return false;
        // Send payload
        size_t sent = 0;
        while (sent < msg.size()) {
            auto n = ::send(fd, msg.data() + sent,
                           msg.size() - sent, 0);
            if (n <= 0) return false;
            sent += n;
        }
        return true;
    }

    static std::string recv_message(int fd) {
        uint32_t len;
        if (::recv(fd, &len, sizeof(len), MSG_WAITALL) != sizeof(len))
            return {};

        std::string msg(len, '\0');
        size_t received = 0;
        while (received < len) {
            auto n = ::recv(fd, msg.data() + received,
                           len - received, 0);
            if (n <= 0) return {};
            received += n;
        }
        return msg;
    }
};

// === Server ===
void server() {
    int srv = socket(AF_UNIX, SOCK_STREAM, 0);
    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, "/tmp/my_ipc.sock", sizeof(addr.sun_path) - 1);
    unlink(addr.sun_path);
    bind(srv, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));
    listen(srv, 5);

    int client = accept(srv, nullptr, nullptr);
    auto msg = SocketIPC::recv_message(client);
    std::cout << "Received: " << msg << "\n";
    SocketIPC::send_message(client, "ACK: " + msg);

    close(client);
    close(srv);
    unlink(addr.sun_path);
}

// === Client ===
void client() {
    int fd = socket(AF_UNIX, SOCK_STREAM, 0);
    sockaddr_un addr{};
    addr.sun_family = AF_UNIX;
    strncpy(addr.sun_path, "/tmp/my_ipc.sock", sizeof(addr.sun_path) - 1);
    connect(fd, reinterpret_cast<sockaddr*>(&addr), sizeof(addr));

    SocketIPC::send_message(fd, "Hello from client");
    auto reply = SocketIPC::recv_message(fd);
    std::cout << "Reply: " << reply << "\n";
    close(fd);
}

```

### Q3: Design a portable IPC abstraction layer

**Answer:**

```cpp

// === IPC Transport abstraction ===
class ITransport {
public:
    virtual ~ITransport() = default;
    virtual bool connect(const std::string& endpoint) = 0;
    virtual bool listen(const std::string& endpoint) = 0;
    virtual bool send(const void* data, size_t len) = 0;
    virtual int recv(void* data, size_t max_len) = 0;
    virtual void close() = 0;
};

// === Message layer on top of transport ===
class IPCChannel {
public:
    explicit IPCChannel(std::unique_ptr<ITransport> transport)
        : transport_(std::move(transport)) {}

    bool send(const std::string& msg) {
        uint32_t len = static_cast<uint32_t>(msg.size());
        if (!transport_->send(&len, sizeof(len))) return false;
        return transport_->send(msg.data(), msg.size());
    }

    std::optional<std::string> receive() {
        uint32_t len;
        if (transport_->recv(&len, sizeof(len)) != sizeof(len))
            return std::nullopt;
        std::string buf(len, '\0');
        if (transport_->recv(buf.data(), len) != static_cast<int>(len))
            return std::nullopt;
        return buf;
    }

private:
    std::unique_ptr<ITransport> transport_;
};

// Concrete transports:
// class TcpTransport : public ITransport { ... };
// class UnixSocketTransport : public ITransport { ... };
// class SharedMemoryTransport : public ITransport { ... };
// class NamedPipeTransport : public ITransport { ... };

// Factory:
std::unique_ptr<ITransport> create_transport(const std::string& type) {
    if (type == "tcp") return std::make_unique<TcpTransport>();
    if (type == "unix") return std::make_unique<UnixSocketTransport>();
    if (type == "shm")  return std::make_unique<SharedMemoryTransport>();
    throw std::runtime_error("Unknown transport: " + type);
}

```

---

## Notes

- **Shared memory** = fastest but requires careful synchronization (atomics or semaphores)
- **Unix domain sockets** = good balance of speed, reliability, and ease of use on Linux/macOS
- **Named pipes** = simplest cross-platform IPC for simple workflows
- Always use **message framing** (length prefix) over stream sockets — `recv()` doesn't guarantee message boundaries
- Clean up IPC resources (`shm_unlink`, `unlink` socket files) on shutdown
- For structured data across IPC, use protobuf/flatbuffers — not raw struct memcpy (alignment, endianness)
- Consider `eventfd`/`signalfd` on Linux for lightweight IPC notification without data
