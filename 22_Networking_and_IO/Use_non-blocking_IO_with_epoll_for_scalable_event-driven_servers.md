# Use non-blocking I/O with epoll for scalable event-driven servers

**Category:** Networking & I/O  
**Item:** #639  
**Reference:** <https://man7.org/linux/man-pages/man7/epoll.7.html>  

---

## Topic Overview

This topic covers building event-driven servers with epoll and non-blocking sockets — complementing #555 (basic epoll multiplexing). This file focuses on event loop architecture, edge-triggered mode, and scalable handler dispatch.

```cpp

epoll event loop architecture:

  Listener socket (non-blocking)
       |  EPOLLIN -> accept()
       v
  +----+-----+-----+-----+
  | fd1 | fd2 | fd3 | ... |  epoll interest list
  +----+-----+-----+-----+
       |
  epoll_wait()  <--- blocks until events
       |
  +-------+-------+-------+
  | event | event | event |  ready list
  +-------+-------+-------+
       |
  dispatch to handlers (read/write/close)

  Edge-triggered (EPOLLET): event fires ONCE when state changes
  Level-triggered (default): event fires as long as condition holds

```

| Mode | Event fires when | Must drain? | Advantage |
| --- | --- | --- | --- |
| Level-triggered (LT) | Data available in buffer | No | Simpler, forgiving |
| Edge-triggered (ET) | NEW data arrives | Yes (EAGAIN) | Fewer epoll_wait wakeups |
| EPOLLONESHOT | Once, then disabled | Re-arm needed | Thread-safe dispatch |

---

## Self-Assessment

### Q1: Set O_NONBLOCK and register with EPOLLET

```cpp

// Linux epoll echo server skeleton
#include <iostream>
#include <cstring>
#include <unordered_map>
#include <vector>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <cerrno>

// Set socket to non-blocking
void set_nonblocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    // Create listener socket
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));

    sockaddr_in addr{};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(listen_fd, (sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, SOMAXCONN);
    set_nonblocking(listen_fd);

    // Create epoll instance
    int epfd = epoll_create1(0);

    // Register listener with edge-triggered mode
    epoll_event ev{};
    ev.events = EPOLLIN | EPOLLET;  // edge-triggered!
    ev.data.fd = listen_fd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    std::cout << "Server listening on port 8080 (EPOLLET mode)\n";

    // Event loop
    constexpr int MAX_EVENTS = 64;
    epoll_event events[MAX_EVENTS];

    for (;;) {
        int nfds = epoll_wait(epfd, events, MAX_EVENTS, -1);

        for (int i = 0; i < nfds; ++i) {
            int fd = events[i].data.fd;

            if (fd == listen_fd) {
                // Accept ALL pending connections (edge-triggered!)
                for (;;) {
                    int client = accept(listen_fd, nullptr, nullptr);
                    if (client < 0) {
                        if (errno == EAGAIN || errno == EWOULDBLOCK)
                            break;  // no more pending
                        perror("accept");
                        break;
                    }
                    set_nonblocking(client);
                    ev.events = EPOLLIN | EPOLLET;
                    ev.data.fd = client;
                    epoll_ctl(epfd, EPOLL_CTL_ADD, client, &ev);
                    std::cout << "Accepted fd=" << client << '\n';
                }
            } else {
                // Handle client I/O in Q2...
            }
        }
    }

    close(epfd);
    close(listen_fd);
}

```

### Q2: Event loop with epoll_wait and handler dispatch

```cpp

#include <iostream>
#include <cstring>
#include <string>
#include <unordered_map>
#include <sys/epoll.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <fcntl.h>
#include <unistd.h>
#include <cerrno>

void set_nonblocking(int fd) {
    fcntl(fd, F_SETFL, fcntl(fd, F_GETFL) | O_NONBLOCK);
}

// Per-connection state
struct Connection {
    std::string read_buf;
    std::string write_buf;
};

int main() {
    int listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    int opt = 1;
    setsockopt(listen_fd, SOL_SOCKET, SO_REUSEADDR, &opt, sizeof(opt));
    sockaddr_in addr{AF_INET, htons(8080), {INADDR_ANY}, {}};
    bind(listen_fd, (sockaddr*)&addr, sizeof(addr));
    listen(listen_fd, 128);
    set_nonblocking(listen_fd);

    int epfd = epoll_create1(0);
    epoll_event ev{EPOLLIN | EPOLLET, {.fd = listen_fd}};
    epoll_ctl(epfd, EPOLL_CTL_ADD, listen_fd, &ev);

    std::unordered_map<int, Connection> conns;
    epoll_event events[64];

    for (;;) {
        int n = epoll_wait(epfd, events, 64, -1);
        for (int i = 0; i < n; ++i) {
            int fd = events[i].data.fd;
            uint32_t ev_mask = events[i].events;

            if (fd == listen_fd) {
                // --- ACCEPT handler ---
                for (;;) {
                    int c = accept(listen_fd, nullptr, nullptr);
                    if (c < 0) break;
                    set_nonblocking(c);
                    ev = {EPOLLIN | EPOLLET, {.fd = c}};
                    epoll_ctl(epfd, EPOLL_CTL_ADD, c, &ev);
                    conns[c] = {};
                }
            } else if (ev_mask & EPOLLIN) {
                // --- READ handler (drain until EAGAIN) ---
                char buf[4096];
                for (;;) {
                    ssize_t nr = read(fd, buf, sizeof(buf));
                    if (nr > 0) {
                        conns[fd].read_buf.append(buf, nr);
                    } else if (nr == 0) {
                        // Client disconnected
                        epoll_ctl(epfd, EPOLL_CTL_DEL, fd, nullptr);
                        close(fd);
                        conns.erase(fd);
                        goto next_event;
                    } else {
                        if (errno == EAGAIN) break;  // fully drained
                        close(fd); conns.erase(fd);
                        goto next_event;
                    }
                }
                // Echo back: move read data to write buffer
                conns[fd].write_buf += conns[fd].read_buf;
                conns[fd].read_buf.clear();
                // Enable write interest
                ev = {EPOLLIN | EPOLLOUT | EPOLLET, {.fd = fd}};
                epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
            } else if (ev_mask & EPOLLOUT) {
                // --- WRITE handler (flush until EAGAIN) ---
                auto& wb = conns[fd].write_buf;
                while (!wb.empty()) {
                    ssize_t nw = write(fd, wb.data(), wb.size());
                    if (nw > 0) {
                        wb.erase(0, nw);
                    } else {
                        break;  // EAGAIN or error
                    }
                }
                if (wb.empty()) {
                    // All written, remove write interest
                    ev = {EPOLLIN | EPOLLET, {.fd = fd}};
                    epoll_ctl(epfd, EPOLL_CTL_MOD, fd, &ev);
                }
            }
            next_event:;
        }
    }
}

```

### Q3: Level-triggered vs edge-triggered

**Level-triggered (LT) — default:**

- epoll_wait returns the fd as long as data is available.
- If you read only 100 bytes but 1000 are buffered, epoll_wait fires again.
- Simpler to program — partial reads are safe.
- More epoll_wait wakeups under high load.

**Edge-triggered (ET) — EPOLLET:**

- epoll_wait returns the fd only when NEW data arrives.
- You MUST drain the fd (read until EAGAIN), or you'll miss data.
- Fewer wakeups = higher throughput at scale.
- Risk: forgetting to drain causes stalls.

```cpp

Timeline with 2 packets arriving:

  Time:  ----[pkt1]--------[pkt2]---->

  LT:    [wake][wake][wake] [wake][wake]  (fires while data present)
  ET:    [wake]             [wake]        (fires only on new arrival)

  If you don't read all of pkt1:
  LT:    [wake again] - safe, you get another chance
  ET:    [no wake]    - STUCK until pkt2 arrives!

```

**EPOLLONESHOT:**

- Fires once, then fd is disabled in epoll.
- Must re-arm with `epoll_ctl(EPOLL_CTL_MOD)` after handling.
- Safe for multi-threaded event loops (prevents two threads handling same fd).

---

## Notes

- Complementary to #555 (basic epoll multiplexing).
- EPOLLET + non-blocking is the standard pattern for high-performance servers (nginx, Redis).
- For C++ RAII: wrap epoll fd and socket fds in unique_fd (see #556, #731).
- Max epoll size: effectively unlimited (backed by red-black tree in kernel).
- On macOS/BSD: use kqueue instead; on Windows: use IOCP. Asio abstracts all.
