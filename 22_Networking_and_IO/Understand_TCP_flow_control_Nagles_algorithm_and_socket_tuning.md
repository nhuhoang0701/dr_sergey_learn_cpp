# Understand TCP flow control, Nagle's algorithm, and socket tuning

**Category:** Networking & I/O  
**Item:** #644  
**Reference:** <https://man7.org/linux/man-pages/man7/tcp.7.html>  

---

## Topic Overview

TCP flow control and socket tuning are critical for network performance. This topic covers TCP_NODELAY, buffer sizing, and TCP_CORK — complementing #554 (Nagle + delayed ACK interaction).

```cpp

Nagle's algorithm (default ON):
  Wait to send small data until:
    a) Previous ACK received, OR
    b) Enough data to fill MSS (~1460 bytes)

  Good for: bulk transfers (fewer small packets)
  Bad for:  RPC/interactive (adds latency)

TCP_NODELAY: disables Nagle -> send immediately
TCP_CORK:    hold ALL data until uncorked -> send one large segment
SO_SNDBUF/SO_RCVBUF: kernel buffer sizes -> affect throughput

```

| Socket option | Effect | Use case |
| --- | --- | --- |
| TCP_NODELAY | Disable Nagle, send immediately | RPC, gaming, trading |
| TCP_CORK | Hold data until uncorked | HTTP response headers + body |
| SO_SNDBUF | Send buffer size | High-throughput, WAN links |
| SO_RCVBUF | Receive buffer size | High-throughput, WAN links |
| TCP_QUICKACK | Disable delayed ACK | Low-latency responses |

---

## Self-Assessment

### Q1: Disable Nagle with TCP_NODELAY for low-latency RPC

```cpp

// Linux/POSIX example
#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <netinet/in.h>
#include <unistd.h>

void set_tcp_nodelay(int sock) {
    int flag = 1;
    int result = setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &flag, sizeof(flag));
    if (result < 0) perror("TCP_NODELAY");
}

// Why TCP_NODELAY matters for RPC:
//
// Without TCP_NODELAY (Nagle ON):
//   write(sock, header, 8);    // sent immediately (no pending data)
//   write(sock, payload, 100); // HELD! waiting for ACK of header
//   // ... 40ms delay (Nagle + delayed ACK interaction) ...
//   // payload finally sent when ACK arrives
//
// With TCP_NODELAY:
//   write(sock, header, 8);    // sent immediately
//   write(sock, payload, 100); // sent immediately
//   // Total: ~0.1ms (just network RTT)

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    // Enable TCP_NODELAY BEFORE connecting (or after accept)
    set_tcp_nodelay(sock);

    // Verify it's set
    int val = 0;
    socklen_t len = sizeof(val);
    getsockopt(sock, IPPROTO_TCP, TCP_NODELAY, &val, &len);
    std::cout << "TCP_NODELAY: " << (val ? "ON" : "OFF") << '\n';  // ON

    std::cout << "\nWhen to use TCP_NODELAY:\n";
    std::cout << "  - RPC protocols (gRPC, Thrift)\n";
    std::cout << "  - Game servers (player updates)\n";
    std::cout << "  - Financial trading (order submission)\n";
    std::cout << "  - Interactive protocols (SSH, Telnet)\n";
    std::cout << "\nWhen NOT to use TCP_NODELAY:\n";
    std::cout << "  - Bulk file transfer (Nagle helps coalesce)\n";
    std::cout << "  - If you can batch writes yourself (writev)\n";

    close(sock);
}

```

### Q2: SO_SNDBUF and SO_RCVBUF for high-throughput

```cpp

#include <iostream>
#include <sys/socket.h>
#include <netinet/in.h>
#include <unistd.h>

// Buffer size calculation for WAN links:
//   BDP = Bandwidth * Delay (Bandwidth-Delay Product)
//   1 Gbps * 10ms RTT = 1,250,000 bytes = ~1.2 MB
//   Default SO_RCVBUF: ~128 KB (Linux) -> bottleneck!
//   Need SO_RCVBUF >= BDP for full utilization

void configure_buffers(int sock) {
    // Set send buffer to 4 MB
    int sndbuf = 4 * 1024 * 1024;
    setsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, sizeof(sndbuf));

    // Set receive buffer to 4 MB
    int rcvbuf = 4 * 1024 * 1024;
    setsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, sizeof(rcvbuf));

    // Note: Linux doubles the value (for bookkeeping overhead)
    // So setting 4MB will show 8MB when reading back
    socklen_t len = sizeof(sndbuf);
    getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, &len);
    getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, &len);
    std::cout << "Actual SO_SNDBUF: " << sndbuf / 1024 << " KB\n";
    std::cout << "Actual SO_RCVBUF: " << rcvbuf / 1024 << " KB\n";
}

int main() {
    int sock = socket(AF_INET, SOCK_STREAM, 0);

    // Check defaults
    int sndbuf, rcvbuf;
    socklen_t len = sizeof(sndbuf);
    getsockopt(sock, SOL_SOCKET, SO_SNDBUF, &sndbuf, &len);
    getsockopt(sock, SOL_SOCKET, SO_RCVBUF, &rcvbuf, &len);
    std::cout << "Default SO_SNDBUF: " << sndbuf / 1024 << " KB\n";
    std::cout << "Default SO_RCVBUF: " << rcvbuf / 1024 << " KB\n";

    // Set larger buffers
    configure_buffers(sock);

    std::cout << "\nBDP examples:\n";
    std::cout << "  LAN (1Gbps, 0.1ms):  ~12.5 KB (default is fine)\n";
    std::cout << "  WAN (1Gbps, 10ms):   ~1.2 MB (need larger buffer)\n";
    std::cout << "  WAN (1Gbps, 100ms):  ~12.5 MB (need much larger)\n";
    std::cout << "\nLinux auto-tuning: net.ipv4.tcp_rmem = min default max\n";
    std::cout << "  Usually: 4096 131072 6291456 (4KB / 128KB / 6MB)\n";

    close(sock);
}

```

### Q3: TCP_CORK batches writes into full segments

```cpp

#include <iostream>
#include <cstring>
#include <sys/socket.h>
#include <netinet/tcp.h>
#include <unistd.h>

// TCP_CORK: buffer ALL writes until uncorked, then send as one burst.
// Like corking a bottle: nothing comes out until you remove the cork.
//
// Without TCP_CORK:
//   write(header)  -> TCP segment 1 (small)
//   write(body)    -> TCP segment 2
//   write(trailer) -> TCP segment 3 (small)
//   = 3 packets, 2 are tiny (inefficient)
//
// With TCP_CORK:
//   cork(sock)
//   write(header)  -> buffered
//   write(body)    -> buffered
//   write(trailer) -> buffered
//   uncork(sock)   -> ALL sent as 1 or 2 full TCP segments
//   = fewer packets, better throughput

void send_http_response(int sock) {
    // Cork the socket (hold all data)
    int flag = 1;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));

    // Write multiple parts (all buffered)
    const char* status = "HTTP/1.1 200 OK\r\n";
    const char* headers = "Content-Type: text/html\r\nContent-Length: 13\r\n\r\n";
    const char* body = "Hello, World!";

    write(sock, status, strlen(status));    // buffered
    write(sock, headers, strlen(headers));  // buffered
    write(sock, body, strlen(body));        // buffered

    // Uncork: send everything as one burst
    flag = 0;
    setsockopt(sock, IPPROTO_TCP, TCP_CORK, &flag, sizeof(flag));
    // All data now sent in minimum number of TCP segments
}

int main() {
    std::cout << "TCP_CORK vs TCP_NODELAY vs writev:\n\n";
    std::cout << "TCP_NODELAY:\n";
    std::cout << "  - Sends each write() immediately\n";
    std::cout << "  - Good for: RPC (latency-sensitive)\n";
    std::cout << "  - Bad for: many small writes (many tiny packets)\n";
    std::cout << "\nTCP_CORK:\n";
    std::cout << "  - Buffers all writes until uncorked\n";
    std::cout << "  - Good for: HTTP responses (header + body)\n";
    std::cout << "  - Requires cork/uncork bracketing\n";
    std::cout << "\nwritev (scatter-gather):\n";
    std::cout << "  - Sends multiple buffers in one syscall\n";
    std::cout << "  - Best of both worlds: one packet, no cork/uncork\n";
    std::cout << "  - Preferred when all data is ready\n";
    std::cout << "\nBest practice: use writev when possible, TCP_CORK when";
    std::cout << " data arrives asynchronously (e.g., streaming body).\n";
}

```

---

## Notes

- Complementary to #554 (Nagle + delayed ACK 200ms spike).
- TCP_NODELAY and TCP_CORK are mutually exclusive on Linux.
- `SO_SNDBUF`/`SO_RCVBUF` should be set BEFORE `connect()`/`listen()` for best effect.
- Linux auto-tuning (tcp_wmem/tcp_rmem) usually does a good job; only override for WAN.
- Use `ss -tnm` to inspect actual buffer usage on Linux.
