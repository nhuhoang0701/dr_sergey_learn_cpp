# Handle DNS resolution asynchronously without blocking the event loop

**Category:** Networking & I/O  
**Item:** #647  
**Standard:** C++11  
**Reference:** <https://think-async.com/Asio/>  

---

## Topic Overview

DNS resolution via `getaddrinfo()` is a blocking OS call (can take 50ms–5s). In an async event loop (Asio, io_uring, epoll), a single blocking DNS call stalls **all** concurrent I/O operations on that thread. Asio provides `async_resolve` which performs resolution on a background thread and delivers results via the event loop.

```cpp

Blocking DNS (BAD in event loop):
  Thread: [IO] [IO] [getaddrinfo...........] [IO] [IO]
                     ^^^^^^^^^^^^^^^^^^^^^
                     ALL other I/O stalled!

Async DNS (GOOD):
  Main:   [IO] [IO] [IO] [IO] [IO] [IO] [callback: resolved!]
  Worker:        [getaddrinfo...........]

```

| Approach | Blocks event loop? | Thread-safe? | Caching? |
| --- | --- | --- | --- |
| `getaddrinfo()` directly | Yes | Yes (reentrant) | No |
| `std::async(getaddrinfo)` | No (separate thread) | Must sync | Manual |
| Asio `async_resolve` | No | Yes (via strand) | No |
| c-ares library | No (non-blocking) | Configurable | TTL-based |

---

## Self-Assessment

### Q1: Use Asio's async_resolve to resolve a hostname without blocking

```cpp

// Requires: Asio standalone or Boost.Asio
// Compile: g++ -std=c++20 -lpthread -o resolver resolver.cpp
// (For standalone Asio: -DASIO_STANDALONE -I/path/to/asio/include)

#include <asio.hpp>
#include <iostream>
#include <chrono>

int main() {
    asio::io_context io;
    asio::ip::tcp::resolver resolver(io);

    auto start = std::chrono::steady_clock::now();

    // async_resolve does NOT block the io_context thread
    resolver.async_resolve(
        "example.com", "443",
        [&start](const asio::error_code& ec,
                 asio::ip::tcp::resolver::results_type results) {
            auto elapsed = std::chrono::steady_clock::now() - start;
            auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(elapsed);

            if (ec) {
                std::cerr << "Resolve failed: " << ec.message() << '\n';
                return;
            }

            std::cout << "Resolved in " << ms.count() << " ms:\n";
            for (const auto& entry : results) {
                std::cout << "  " << entry.endpoint().address().to_string()
                          << ":" << entry.endpoint().port() << '\n';
            }
        }
    );

    // Other async operations can proceed concurrently
    asio::steady_timer timer(io, std::chrono::milliseconds(1));
    timer.async_wait([](const asio::error_code&) {
        std::cout << "Timer fired (not blocked by DNS!)\n";
    });

    io.run();  // drives both timer and DNS resolution
    // Output:
    //   Timer fired (not blocked by DNS!)
    //   Resolved in 45 ms:
    //     93.184.216.34:443
    //     2606:2800:220:1:248:1893:25c8:1946:443
}

```

### Q2: The getaddrinfo blocking danger

```cpp

#include <iostream>
#include <chrono>
#include <thread>
#include <cstring>

#ifdef _WIN32
#include <winsock2.h>
#include <ws2tcpip.h>
#pragma comment(lib, "ws2_32.lib")
#else
#include <netdb.h>
#include <sys/socket.h>
#endif

// DANGER: getaddrinfo is blocking!
void resolve_blocking(const char* host) {
    auto t0 = std::chrono::steady_clock::now();

    struct addrinfo hints{};
    hints.ai_family = AF_UNSPEC;      // IPv4 or IPv6
    hints.ai_socktype = SOCK_STREAM;  // TCP
    struct addrinfo* result = nullptr;

    int err = getaddrinfo(host, "443", &hints, &result);
    // ^^ THIS BLOCKS THE CALLING THREAD!
    // If DNS server is slow or unreachable: can block 5-30 SECONDS

    auto t1 = std::chrono::steady_clock::now();
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(t1 - t0);

    if (err != 0) {
        std::cerr << "getaddrinfo failed: " << gai_strerror(err)
                  << " (" << ms.count() << " ms)\n";
        return;
    }

    std::cout << "Resolved " << host << " in " << ms.count() << " ms\n";
    for (auto rp = result; rp; rp = rp->ai_next) {
        char addr[64];
        if (rp->ai_family == AF_INET) {
            auto* sin = reinterpret_cast<struct sockaddr_in*>(rp->ai_addr);
            inet_ntop(AF_INET, &sin->sin_addr, addr, sizeof(addr));
        } else {
            auto* sin6 = reinterpret_cast<struct sockaddr_in6*>(rp->ai_addr);
            inet_ntop(AF_INET6, &sin6->sin6_addr, addr, sizeof(addr));
        }
        std::cout << "  " << addr << '\n';
    }
    freeaddrinfo(result);
}

int main() {
    // In an event loop, this would stall ALL concurrent connections:
    //   epoll_wait -> getaddrinfo (BLOCKS 50ms-30s!) -> missed events!
    resolve_blocking("example.com");
    resolve_blocking("nonexistent.invalid");  // may block 5+ seconds!

    std::cout << "\nSafe alternatives for event loops:\n";
    std::cout << "  1. Asio async_resolve (resolves on thread pool)\n";
    std::cout << "  2. c-ares library (truly non-blocking, uses select/epoll)\n";
    std::cout << "  3. std::async + getaddrinfo (DIY thread offload)\n";
    std::cout << "  4. getaddrinfo_a (glibc only, POSIX async DNS)\n";
}

```

### Q3: Implement a DNS cache

```cpp

#include <iostream>
#include <string>
#include <unordered_map>
#include <vector>
#include <chrono>
#include <mutex>
#include <optional>

// Simple thread-safe DNS cache with TTL
class DnsCache {
public:
    struct Entry {
        std::vector<std::string> addresses;
        std::chrono::steady_clock::time_point expires;
    };

    // Check cache before resolving
    std::optional<std::vector<std::string>> lookup(const std::string& host) {
        std::lock_guard<std::mutex> lock(mu_);
        auto it = cache_.find(host);
        if (it == cache_.end()) return std::nullopt;

        if (std::chrono::steady_clock::now() > it->second.expires) {
            cache_.erase(it);  // expired
            return std::nullopt;
        }
        return it->second.addresses;
    }

    // Store resolved addresses with TTL
    void store(const std::string& host,
               std::vector<std::string> addrs,
               std::chrono::seconds ttl = std::chrono::seconds(300)) {
        std::lock_guard<std::mutex> lock(mu_);
        cache_[host] = Entry{
            std::move(addrs),
            std::chrono::steady_clock::now() + ttl
        };
    }

    void evict_expired() {
        std::lock_guard<std::mutex> lock(mu_);
        auto now = std::chrono::steady_clock::now();
        for (auto it = cache_.begin(); it != cache_.end(); ) {
            if (now > it->second.expires)
                it = cache_.erase(it);
            else
                ++it;
        }
    }

private:
    std::mutex mu_;
    std::unordered_map<std::string, Entry> cache_;
};

int main() {
    DnsCache cache;

    // Simulate: first lookup -> miss -> resolve -> store
    auto result = cache.lookup("example.com");
    std::cout << "First lookup: " << (result ? "HIT" : "MISS") << '\n';
    // MISS -> would call async_resolve here

    // Store result
    cache.store("example.com", {"93.184.216.34", "2606:2800:220:1::248"});

    // Second lookup -> hit
    result = cache.lookup("example.com");
    std::cout << "Second lookup: " << (result ? "HIT" : "MISS") << '\n';
    if (result) {
        for (const auto& addr : *result)
            std::cout << "  " << addr << '\n';
    }

    // Integration with Asio:
    // resolver.async_resolve(host, port,
    //     [&cache, host](auto ec, auto results) {
    //         if (!ec) {
    //             std::vector<std::string> addrs;
    //             for (auto& r : results)
    //                 addrs.push_back(r.endpoint().address().to_string());
    //             cache.store(host, std::move(addrs));
    //         }
    //     });

    // Output:
    //   First lookup: MISS
    //   Second lookup: HIT
    //     93.184.216.34
    //     2606:2800:220:1::248
}

```

---

## Notes

- `getaddrinfo` timeout depends on OS (Linux: ~5s per nameserver * retries).
- Asio's `async_resolve` uses a thread pool internally — resolution is OS-blocking but off the event thread.
- c-ares provides truly non-blocking DNS using its own protocol implementation.
- DNS TTL from the server should ideally guide cache expiration (300s is a common default).
- For high-traffic servers, DNS caching prevents thundering herd on reconnection storms.
