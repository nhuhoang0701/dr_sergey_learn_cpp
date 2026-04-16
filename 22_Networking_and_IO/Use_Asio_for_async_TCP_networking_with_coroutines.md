# Use Asio for async TCP networking with coroutines

**Category:** Networking & I/O  
**Item:** #550  
**Standard:** C++20  
**Reference:** <https://think-async.com/Asio/>  

---

## Topic Overview

This topic focuses on the io_context event loop mechanics and implementing timeouts with `steady_timer` — complementing #643 (client + error handling) and #729 (server + thread pool).

```cpp

asio::io_context lifecycle:
  io_context io;              // create the event loop
  co_spawn(io, coroutine());  // register work
  io.run();                   // drives the loop until no work remains

  io.run() flow:
    while (has_pending_operations) {
      event = epoll_wait/IOCP();    // wait for I/O or timer
      resume_coroutine(event);      // resume the suspended coroutine
    }

```

| io_context method | Behavior |
| --- | --- |
| `run()` | Blocks until all work done |
| `run_one()` | Processes one event, then returns |
| `poll()` | Non-blocking: processes ready events only |
| `stop()` | Signals run() to exit |
| `restart()` | Allows run() to be called again after stop |

---

## Self-Assessment

### Q1: Async echo server with co_await

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::ip::tcp;
using asio::use_awaitable;

asio::awaitable<void> echo_session(tcp::socket sock) {
    char buf[1024];
    try {
        for (;;) {
            // async_read_some: read whatever is available
            size_t n = co_await sock.async_read_some(
                asio::buffer(buf), use_awaitable);

            // async_write: write all bytes (handles partial writes)
            co_await asio::async_write(
                sock, asio::buffer(buf, n), use_awaitable);
        }
    } catch (const asio::system_error& e) {
        if (e.code() == asio::error::eof)
            std::cout << "Client disconnected\n";
        else
            std::cerr << "Error: " << e.what() << '\n';
    }
}

asio::awaitable<void> accept_loop(asio::io_context& io, uint16_t port) {
    tcp::acceptor acc(io, {tcp::v4(), port});
    for (;;) {
        auto sock = co_await acc.async_accept(use_awaitable);
        asio::co_spawn(io, echo_session(std::move(sock)), asio::detached);
    }
}

int main() {
    asio::io_context io;
    asio::co_spawn(io, accept_loop(io, 8080), asio::detached);
    std::cout << "Echo server on :8080\n";
    io.run();
}

```

### Q2: io_context::run() drives the event loop

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::use_awaitable;

asio::awaitable<void> task1(asio::io_context& io) {
    asio::steady_timer t(io, std::chrono::seconds(1));
    co_await t.async_wait(use_awaitable);
    std::cout << "Task 1 complete\n";
}

int main() {
    asio::io_context io;

    // Without co_spawn/post: run() returns immediately (no work!)
    std::cout << "run() without work:\n";
    io.run();  // returns immediately
    std::cout << "  returned (no work)\n";
    io.restart();  // needed before calling run() again

    // With work: run() blocks until coroutine finishes
    std::cout << "run() with coroutine:\n";
    asio::co_spawn(io, task1(io), asio::detached);
    io.run();  // blocks ~1 second, then returns
    std::cout << "  returned (coroutine done)\n";
    io.restart();

    // With work guard: run() never returns (for servers)
    // auto guard = asio::make_work_guard(io);
    // io.run();  // blocks forever (even with no pending ops)
    // guard.reset();  // allow run() to exit

    std::cout << "\nio_context::run() internals:\n";
    std::cout << "  1. Check for pending async operations\n";
    std::cout << "  2. If none: return (or block if work_guard exists)\n";
    std::cout << "  3. Wait for events (epoll_wait / IOCP)\n";
    std::cout << "  4. Resume coroutines whose I/O completed\n";
    std::cout << "  5. Repeat from step 1\n";
}

```

### Q3: Connection timeout with steady_timer

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <asio/experimental/awaitable_operators.hpp>
#include <iostream>

using asio::ip::tcp;
using asio::use_awaitable;
using namespace asio::experimental::awaitable_operators;

// Timeout pattern: race async_connect against a timer
asio::awaitable<void> connect_with_timeout(
    asio::io_context& io,
    const std::string& host, const std::string& port,
    std::chrono::seconds timeout)
{
    tcp::resolver resolver(io);
    auto endpoints = co_await resolver.async_resolve(
        host, port, use_awaitable);

    tcp::socket socket(io);

    // Method 1: Manual timer + cancel
    asio::steady_timer timer(io, timeout);

    // Start both: connect and timer
    bool timed_out = false;
    timer.async_wait([&socket, &timed_out](asio::error_code ec) {
        if (!ec) {  // timer fired (not cancelled)
            timed_out = true;
            socket.close();  // cancel pending connect
        }
    });

    try {
        co_await asio::async_connect(socket, endpoints, use_awaitable);
        timer.cancel();  // connected: cancel the timer
        std::cout << "Connected within timeout!\n";

        // Use the socket...
        co_await asio::async_write(
            socket, asio::buffer("Hello\n", 6), use_awaitable);

    } catch (const asio::system_error& e) {
        if (timed_out) {
            std::cout << "Connection timed out after "
                      << timeout.count() << "s\n";
        } else {
            std::cerr << "Connect error: " << e.what() << '\n';
        }
    }
}

// Read with per-operation timeout
asio::awaitable<void> read_with_timeout(
    tcp::socket& socket, asio::io_context& io) {
    char buf[1024];

    // Timeout for each read operation
    for (;;) {
        asio::steady_timer timer(io, std::chrono::seconds(30));
        bool timed_out = false;

        timer.async_wait([&](asio::error_code ec) {
            if (!ec) {
                timed_out = true;
                socket.close();
            }
        });

        try {
            auto n = co_await socket.async_read_some(
                asio::buffer(buf), use_awaitable);
            timer.cancel();
            std::cout << "Read " << n << " bytes\n";
        } catch (...) {
            if (timed_out)
                std::cout << "Read timed out\n";
            break;
        }
    }
}

int main() {
    asio::io_context io;
    asio::co_spawn(io,
        connect_with_timeout(io, "example.com", "80", std::chrono::seconds(5)),
        asio::detached);
    io.run();
}

```

---

## Notes

- Complementary to #643 (client + error handling) and #729 (server + thread pool).
- `io_context::run()` must be called to actually execute any async work.
- `make_work_guard(io)` prevents run() from returning when all work is done (use for servers).
- Timeout pattern: start timer + async op; whichever completes first cancels the other.
- Asio 1.28+: `asio::experimental::awaitable_operators` provides `operator||` for racing awaitables.
