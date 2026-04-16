# Wrap POSIX sockets in RAII for safe resource management

**Category:** Networking & I/O  
**Item:** #549  
**Reference:** <https://man7.org/linux/man-pages/man2/socket.2.html>  

---

## Topic Overview

This topic focuses on move-only semantics, socket options, and EINTR handling in RAII socket wrappers — complementing #726 (basic RAII + TCP client + leak prevention) and #638 (TCP server with unique_ptr).

```cpp

Move-only socket ownership:

  Socket a(AF_INET, SOCK_STREAM);   // a owns fd=3
  Socket b = std::move(a);           // b owns fd=3, a.fd_=-1
  // Socket c = b;                   // COMPILE ERROR: deleted copy

  EINTR retry pattern:
  ssize_t n;
  do {
      n = ::recv(fd, buf, len, 0);
  } while (n < 0 && errno == EINTR);  // signal interrupted, retry

```

| Socket option | Level | What it does |
| --- | --- | --- |
| SO_REUSEADDR | SOL_SOCKET | Allow bind to port in TIME_WAIT |
| SO_REUSEPORT | SOL_SOCKET | Multiple sockets bind to same port |
| TCP_NODELAY | IPPROTO_TCP | Disable Nagle (send immediately) |
| SO_KEEPALIVE | SOL_SOCKET | Detect dead connections |
| SO_RCVBUF/SO_SNDBUF | SOL_SOCKET | Set buffer sizes |
| SO_LINGER | SOL_SOCKET | Control close() behavior |

---

## Self-Assessment

### Q1: Move-only RAII Socket class

```cpp

#include <iostream>
#include <utility>
#include <cstring>
#include <sys/socket.h>
#include <unistd.h>

class Socket {
public:
    // Create new socket
    Socket(int domain, int type, int protocol = 0)
        : fd_(::socket(domain, type, protocol)) {
        if (fd_ < 0)
            throw std::runtime_error(strerror(errno));
    }

    // Adopt existing fd
    static Socket from_fd(int fd) { return Socket(fd, adopt_tag{}); }

    // Destructor
    ~Socket() { close(); }

    // Move constructor
    Socket(Socket&& other) noexcept : fd_(other.fd_) {
        other.fd_ = -1;
    }

    // Move assignment
    Socket& operator=(Socket&& other) noexcept {
        if (this != &other) {
            close();           // close current fd
            fd_ = other.fd_;   // take ownership
            other.fd_ = -1;    // release source
        }
        return *this;
    }

    // Delete copy (you cannot duplicate a file descriptor safely)
    Socket(const Socket&) = delete;
    Socket& operator=(const Socket&) = delete;

    int fd() const { return fd_; }
    bool valid() const { return fd_ >= 0; }
    explicit operator bool() const { return valid(); }

private:
    struct adopt_tag {};
    Socket(int fd, adopt_tag) : fd_(fd) {}

    void close() {
        if (fd_ >= 0) {
            ::close(fd_);
            fd_ = -1;
        }
    }

    int fd_ = -1;
};

int main() {
    Socket a(AF_INET, SOCK_STREAM);
    std::cout << "a.fd=" << a.fd() << '\n';

    Socket b = std::move(a);  // move ownership
    std::cout << "After move: a.fd=" << a.fd()
              << " b.fd=" << b.fd() << '\n';

    // Socket c = b;  // ERROR: copy deleted

    b = Socket(AF_INET, SOCK_STREAM);  // move-assign: old fd closed
    std::cout << "After reassign: b.fd=" << b.fd() << '\n';
}
// Output:
//   a.fd=3
//   After move: a.fd=-1 b.fd=3
//   After reassign: b.fd=4

```

### Q2: Socket options through the wrapper

```cpp

#include <iostream>
#include <cstring>
#include <stdexcept>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/tcp.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }

    // Generic setsockopt wrapper
    template<typename T>
    void set_option(int level, int optname, const T& value) {
        if (setsockopt(fd_, level, optname, &value, sizeof(value)) < 0)
            throw std::runtime_error(
                std::string("setsockopt: ") + strerror(errno));
    }

    template<typename T>
    T get_option(int level, int optname) const {
        T value{};
        socklen_t len = sizeof(value);
        if (getsockopt(fd_, level, optname, &value, &len) < 0)
            throw std::runtime_error(strerror(errno));
        return value;
    }

    // Typed convenience methods
    void set_reuse_addr(bool on = true) {
        int val = on ? 1 : 0;
        set_option(SOL_SOCKET, SO_REUSEADDR, val);
    }

    void set_tcp_nodelay(bool on = true) {
        int val = on ? 1 : 0;
        set_option(IPPROTO_TCP, TCP_NODELAY, val);
    }

    void set_recv_buffer(int size) {
        set_option(SOL_SOCKET, SO_RCVBUF, size);
    }

    void set_keepalive(bool on = true) {
        int val = on ? 1 : 0;
        set_option(SOL_SOCKET, SO_KEEPALIVE, val);
    }

private:
    int fd_;
};

int main() {
    Socket sock(AF_INET, SOCK_STREAM);

    // Set options safely through the wrapper
    sock.set_reuse_addr();      // SO_REUSEADDR=1
    sock.set_tcp_nodelay();     // TCP_NODELAY=1 (disable Nagle)
    sock.set_recv_buffer(65536);  // SO_RCVBUF=64K
    sock.set_keepalive();       // SO_KEEPALIVE=1

    // Verify
    int nodelay = sock.get_option<int>(IPPROTO_TCP, TCP_NODELAY);
    int rcvbuf  = sock.get_option<int>(SOL_SOCKET, SO_RCVBUF);

    std::cout << "TCP_NODELAY: " << nodelay << '\n';
    std::cout << "SO_RCVBUF: "   << rcvbuf << '\n';
}
// Output:
//   TCP_NODELAY: 1
//   SO_RCVBUF: 131072  (kernel doubles the requested size)

```

### Q3: EINTR retry loop

```cpp

#include <iostream>
#include <cstring>
#include <cerrno>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

class Socket {
public:
    Socket(int domain, int type, int proto = 0)
        : fd_(::socket(domain, type, proto)) {}
    ~Socket() { if (fd_ >= 0) ::close(fd_); }
    Socket(Socket&& o) noexcept : fd_(o.fd_) { o.fd_ = -1; }
    Socket(const Socket&) = delete;
    int fd() const { return fd_; }

    // Send with EINTR retry
    ssize_t send_all(const void* data, size_t len) {
        const char* ptr = static_cast<const char*>(data);
        size_t remaining = len;

        while (remaining > 0) {
            ssize_t n = ::send(fd_, ptr, remaining, 0);
            if (n < 0) {
                if (errno == EINTR) continue;  // signal interrupted, retry!
                return -1;  // real error
            }
            ptr += n;
            remaining -= n;
        }
        return len;
    }

    // Recv with EINTR retry
    ssize_t recv_some(void* buf, size_t len) {
        ssize_t n;
        do {
            n = ::recv(fd_, buf, len, 0);
        } while (n < 0 && errno == EINTR);  // retry on signal
        return n;
    }

    // Accept with EINTR retry
    Socket accept() {
        int client_fd;
        do {
            client_fd = ::accept(fd_, nullptr, nullptr);
        } while (client_fd < 0 && errno == EINTR);

        if (client_fd < 0)
            throw std::runtime_error(strerror(errno));
        return Socket(client_fd);
    }

private:
    Socket(int fd) : fd_(fd) {}
    int fd_;
};

int main() {
    std::cout << "EINTR handling patterns:\n";
    std::cout << "  recv: do { n = recv(); } while (n<0 && errno==EINTR);\n";
    std::cout << "  send: if (n<0 && errno==EINTR) continue;\n";
    std::cout << "  accept: do { fd = accept(); } while (fd<0 && errno==EINTR);\n";
    std::cout << "\nWhy EINTR happens:\n";
    std::cout << "  A signal (SIGCHLD, SIGALRM, etc.) interrupts a blocking syscall.\n";
    std::cout << "  The syscall returns -1 with errno=EINTR.\n";
    std::cout << "  Solution: retry the syscall (it's safe).\n";
    std::cout << "  SA_RESTART flag makes some syscalls auto-restart,\n";
    std::cout << "  but NOT all (accept, select, poll don't auto-restart).\n";
}

```

---

## Notes

- Complementary to #726 (basic RAII + TCP client) and #638 (TCP server with unique_ptr).
- Always handle EINTR in production code, especially on servers with signal handlers.
- `SO_REUSEADDR` is essential for servers: without it, bind() fails for ~60s after restart (TIME_WAIT).
- TCP_NODELAY is critical for interactive protocols (SSH, HTTP/2); harmful for bulk transfers.
- Consider `SO_LINGER` with timeout=0 for immediate close (sends RST instead of FIN).
