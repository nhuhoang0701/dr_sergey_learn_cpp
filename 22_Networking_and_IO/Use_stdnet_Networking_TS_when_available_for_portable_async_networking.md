# Use std::net (Networking TS) when available for portable async networking

**Category:** Networking & I/O  
**Item:** #731  
**Standard:** C++26 (proposed)  
**Reference:** <https://cplusplus.github.io/networking-ts/draft.pdf>  

---

## Topic Overview

The Networking TS (Technical Specification, N4771) proposes a standard async networking API heavily based on Asio. While not yet in the standard, Asio serves as the reference implementation. The API surface: `io_context`, `tcp::acceptor`, `tcp::socket`, `async_read`, `async_write`.

```cpp

Networking TS architecture:

  io_context (event loop)            Proactor pattern
  +------------------+              +-------------------+
  | run()            | <----------- | completion handler|
  | poll()           |   callback   | or coroutine      |
  | stop()           |              +-------------------+
  +------------------+
        |
  +-----+------+------+
  | acceptor   | timer | socket |   I/O objects
  | async_     | async_| async_ |
  | accept()   | wait()| read() |
  +-----+------+------+--------+

  Relationship to std::execution:
  Networking TS:   io_context = scheduler, async ops = senders
  Proposed: integrate with sender/receiver (P2300)

```

| Component | Networking TS | Asio equivalent |
| --- | --- | --- |
| Event loop | `std::net::io_context` | `asio::io_context` |
| TCP acceptor | `std::net::ip::tcp::acceptor` | `asio::ip::tcp::acceptor` |
| TCP socket | `std::net::ip::tcp::socket` | `asio::ip::tcp::socket` |
| Endpoint | `std::net::ip::tcp::endpoint` | `asio::ip::tcp::endpoint` |
| Async read | `std::net::async_read()` | `asio::async_read()` |
| Buffer | `std::net::buffer()` | `asio::buffer()` |

---

## Self-Assessment

### Q1: Networking TS API surface

**Core types and their roles:**

**`io_context`** — The event loop / scheduler:

- Runs async operations to completion.
- `run()`: blocks and processes handlers until no work remains.
- `poll()`: runs ready handlers without blocking.
- `stop()`: interrupts `run()`.
- Thread-safe: multiple threads can call `run()` concurrently.

**`tcp::acceptor`** — Listens for incoming connections:

- `open()`, `bind()`, `listen()` — or construct with endpoint.
- `accept()` / `async_accept()` — returns a connected socket.

**`tcp::socket`** — A connected TCP stream:

- `connect()` / `async_connect()` — initiate outbound connection.
- `read_some()` / `async_read_some()` — read available data.
- `write_some()` / `async_write_some()` — write data.
- Composed operations: `async_read()`, `async_write()` read/write exact amounts.

**`tcp::endpoint`** — IP address + port:

- `tcp::endpoint(ip::address_v4::any(), 8080)` — listen on all interfaces.
- `tcp::resolver` — DNS resolution (also async).

```cpp

// Using Asio as Networking TS reference implementation
// Compile: g++ -std=c++20 -I/path/to/asio/include -DASIO_STANDALONE net_ts.cpp
#include <asio.hpp>
#include <iostream>

int main() {
    // io_context: the event loop
    asio::io_context ctx;

    // tcp::acceptor: listen on port 8080
    asio::ip::tcp::acceptor acceptor(
        ctx, {asio::ip::tcp::v4(), 8080});

    std::cout << "Listening on port 8080...\n";

    // Accept one connection (synchronous)
    asio::ip::tcp::socket sock(ctx);
    acceptor.accept(sock);

    std::cout << "Connected: " << sock.remote_endpoint() << '\n';

    // Read some data
    char buf[256];
    std::error_code ec;
    size_t n = sock.read_some(asio::buffer(buf), ec);
    if (!ec) {
        std::cout << "Received: ";
        std::cout.write(buf, n);
    }

    // Write response
    asio::write(sock, asio::buffer("Hello from Networking TS!\n"));
}

```

### Q2: Minimal TCP echo server

```cpp

// Async echo server using Asio (Networking TS reference)
#include <asio.hpp>
#include <iostream>
#include <memory>

using asio::ip::tcp;

class Session : public std::enable_shared_from_this<Session> {
public:
    explicit Session(tcp::socket socket)
        : socket_(std::move(socket)) {}

    void start() { do_read(); }

private:
    void do_read() {
        auto self = shared_from_this();
        socket_.async_read_some(
            asio::buffer(buf_),
            [self](std::error_code ec, size_t len) {
                if (!ec) self->do_write(len);
            });
    }

    void do_write(size_t len) {
        auto self = shared_from_this();
        asio::async_write(
            socket_, asio::buffer(buf_, len),
            [self](std::error_code ec, size_t) {
                if (!ec) self->do_read();  // echo loop
            });
    }

    tcp::socket socket_;
    char buf_[1024];
};

class Server {
public:
    Server(asio::io_context& ctx, unsigned short port)
        : acceptor_(ctx, {tcp::v4(), port}) {
        do_accept();
    }

private:
    void do_accept() {
        acceptor_.async_accept(
            [this](std::error_code ec, tcp::socket socket) {
                if (!ec) {
                    std::make_shared<Session>(std::move(socket))->start();
                }
                do_accept();  // accept next connection
            });
    }

    tcp::acceptor acceptor_;
};

int main() {
    asio::io_context ctx;
    Server server(ctx, 8080);
    std::cout << "Echo server on port 8080\n";
    ctx.run();  // blocks, processes all async ops
}
// Test: echo "hello" | nc localhost 8080

```

### Q3: Networking TS and std::execution senders

The Networking TS and `std::execution` (P2300 senders/receivers) are converging:

**Current Networking TS (callback-based):**

```cpp

socket.async_read_some(buffer, [](error_code ec, size_t n) {
    // completion handler (callback)
});

```

**Future integration with senders:**

```cpp

// Proposed: async operations return senders
auto sender = socket.async_read_some(buffer, asio::use_sender);
// Can be composed with std::execution algorithms:
auto pipeline = std::execution::then(sender, [](size_t n) {
    return process(n);
});
std::execution::sync_wait(pipeline);

```

**Key connections:**

| Concept | Networking TS | std::execution |
| --- | --- | --- |
| Event loop | `io_context` | `run_loop` / scheduler |
| Async operation | `async_read()` + callback | sender |
| Composition | Nested callbacks / coroutines | `then`, `when_all`, `let_value` |
| Cancellation | `socket.cancel()` | `stop_token` |
| Error handling | `error_code` parameter | `set_error` channel |

**Current status (2024):**

- Networking TS is NOT in C++23 or C++26.
- `std::execution` (P2300) is in C++26.
- The plan: redefine networking async ops as senders.
- Use Asio now; it already supports `asio::use_awaitable` (coroutines) and experimental sender support.

---

## Notes

- Asio IS the reference implementation for Networking TS. Use it now.
- `asio::use_awaitable` turns any async op into a coroutine awaitable (C++20).
- `asio::deferred` creates composable async operations.
- For production: Asio, Boost.Beast (HTTP/WebSocket on Asio), or libcurl.
- The Networking TS delay is because the committee wants sender/receiver integration first.
