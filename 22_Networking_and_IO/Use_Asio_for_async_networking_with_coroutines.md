# Use Asio for async networking with coroutines

**Category:** Networking & I/O  
**Item:** #729  
**Standard:** C++20  
**Reference:** <https://think-async.com/Asio/>  

---

## Topic Overview

This topic covers building an async TCP echo server with Asio coroutines and scaling it with a thread pool — complementing #643 (client + error handling) and #550 (echo server + timeouts).

```cpp

Asio coroutine echo server architecture:
  co_spawn(io, listener())          // accept loop
    |-> co_spawn(io, session(sock)) // per-client coroutine
    |-> co_spawn(io, session(sock))
    |-> ...                         // thousands of concurrent sessions

All sessions run on io.run() thread(s) cooperatively.
No thread-per-connection overhead.

```

| Concurrency model | Threads | Connections | Complexity |
| --- | --- | --- | --- |
| Thread per connection | 1:1 | Limited (~1K) | Low |
| Callback-based async | Pool | High (~100K) | High (callback hell) |
| Coroutine-based async | Pool | High (~100K) | Low (sequential) |

---

## Self-Assessment

### Q1: Async TCP echo server with Asio coroutines

```cpp

// Requires: standalone Asio or Boost.Asio
// Compile: g++ -std=c++20 -fcoroutines -I/path/to/asio -lpthread server.cpp
#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::ip::tcp;
using asio::use_awaitable;

// Per-client session coroutine
asio::awaitable<void> session(tcp::socket socket) {
    try {
        char buf[1024];
        for (;;) {
            // co_await suspends until data arrives
            size_t n = co_await socket.async_read_some(
                asio::buffer(buf), use_awaitable);

            // Echo back
            co_await asio::async_write(
                socket, asio::buffer(buf, n), use_awaitable);
        }
    } catch (const asio::system_error& e) {
        if (e.code() != asio::error::eof)
            std::cerr << "Session error: " << e.what() << '\n';
    }
    // Socket destructor closes connection (RAII)
}

// Listener coroutine: accepts connections indefinitely
asio::awaitable<void> listener(asio::io_context& io, uint16_t port) {
    tcp::acceptor acceptor(io, {tcp::v4(), port});
    std::cout << "Listening on port " << port << '\n';

    for (;;) {
        // co_await suspends until a new connection arrives
        tcp::socket socket = co_await acceptor.async_accept(use_awaitable);
        std::cout << "Client connected: "
                  << socket.remote_endpoint() << '\n';

        // Spawn a new session coroutine for this client
        // Multiple sessions run concurrently on the same thread!
        asio::co_spawn(io, session(std::move(socket)), asio::detached);
    }
}

int main() {
    asio::io_context io;
    asio::co_spawn(io, listener(io, 8080), asio::detached);
    io.run();  // single-threaded event loop
}
// Test: echo "Hello" | nc localhost 8080

```

### Q2: Coroutines look sequential but don't block

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::use_awaitable;

// This coroutine "looks" synchronous but is fully async:
asio::awaitable<void> demo(asio::io_context& io) {
    // Step 1: timer (1 second)
    asio::steady_timer t1(io, std::chrono::seconds(1));
    std::cout << "Starting timer 1...\n";
    co_await t1.async_wait(use_awaitable);  // SUSPENDS, doesn't block!
    std::cout << "Timer 1 done!\n";

    // Step 2: another timer
    asio::steady_timer t2(io, std::chrono::milliseconds(500));
    std::cout << "Starting timer 2...\n";
    co_await t2.async_wait(use_awaitable);  // SUSPENDS again
    std::cout << "Timer 2 done!\n";

    // This reads like synchronous code:
    //   sleep(1); print; sleep(0.5); print;
    // But the thread is FREE during co_await!
    // Other coroutines can run during the waits.
}

// Prove: multiple coroutines interleave on ONE thread
asio::awaitable<void> task_a(asio::io_context& io) {
    for (int i = 0; i < 3; ++i) {
        asio::steady_timer t(io, std::chrono::milliseconds(100));
        co_await t.async_wait(use_awaitable);
        std::cout << "A" << i << " ";
    }
}

asio::awaitable<void> task_b(asio::io_context& io) {
    for (int i = 0; i < 3; ++i) {
        asio::steady_timer t(io, std::chrono::milliseconds(100));
        co_await t.async_wait(use_awaitable);
        std::cout << "B" << i << " ";
    }
}

int main() {
    asio::io_context io;

    // Spawn two coroutines on the SAME thread
    asio::co_spawn(io, task_a(io), asio::detached);
    asio::co_spawn(io, task_b(io), asio::detached);

    io.run();  // single thread drives both!
    std::cout << "\n";
    // Output interleaved: A0 B0 A1 B1 A2 B2
    // (or similar, depending on timer resolution)
}

```

### Q3: Thread pool executor for multi-core scaling

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>
#include <thread>

using asio::ip::tcp;
using asio::use_awaitable;

asio::awaitable<void> session(tcp::socket socket) {
    try {
        char buf[4096];
        for (;;) {
            auto n = co_await socket.async_read_some(
                asio::buffer(buf), use_awaitable);
            co_await asio::async_write(
                socket, asio::buffer(buf, n), use_awaitable);
        }
    } catch (...) {}
}

asio::awaitable<void> listener(tcp::acceptor& acceptor) {
    for (;;) {
        auto socket = co_await acceptor.async_accept(use_awaitable);
        auto exec = co_await asio::this_coro::executor;
        asio::co_spawn(exec, session(std::move(socket)), asio::detached);
    }
}

int main() {
    // Thread pool: N threads running io_context::run()
    unsigned n_threads = std::thread::hardware_concurrency();
    asio::io_context io(n_threads);

    tcp::acceptor acceptor(io, {tcp::v4(), 8080});
    asio::co_spawn(io, listener(acceptor), asio::detached);

    std::cout << "Echo server on :8080 with " << n_threads << " threads\n";

    // Launch thread pool
    std::vector<std::thread> threads;
    for (unsigned i = 1; i < n_threads; ++i)
        threads.emplace_back([&io] { io.run(); });

    io.run();  // main thread also participates

    for (auto& t : threads) t.join();
}
// All threads call io.run() -> coroutines can be resumed on ANY thread
// Asio uses implicit strand for each socket (safe without explicit locking)
// Throughput scales linearly with core count for independent sessions

```

---

## Notes

- Complementary to #643 (client + error handling) and #550 (echo server + timeouts).
- Each `co_spawn` creates one coroutine frame on the heap (~200-500 bytes).
- 100K concurrent connections = ~50MB coroutine memory (vs ~800MB for 100K threads).
- For shared state across sessions: use `asio::strand` to serialize access.
- Consider `asio::experimental::channel` for inter-coroutine communication.
