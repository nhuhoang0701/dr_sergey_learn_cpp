# Use Asio coroutines for async TCP client/server

**Category:** Networking & I/O  
**Item:** #643  
**Standard:** C++20  
**Reference:** <https://think-async.com/Asio/asio-1.28.0/doc/asio/overview/composition/cpp20_coroutines.html>  

---

## Topic Overview

Asio C++20 coroutines combine async I/O with `co_await`, producing code that reads sequentially but runs asynchronously. This topic focuses on the client side and error handling — complementing #729 (server + thread pool) and #550 (echo server + timers).

```cpp

Callback-based (hard to follow):
  async_connect(ep, [](auto ec) {
      if (ec) return;
      async_write(sock, buf, [](auto ec, auto n) {
          if (ec) return;
          async_read(sock, buf, [](auto ec, auto n) {
              // callback hell!
          });
      });
  });

Coroutine-based (reads like synchronous code):
  co_await async_connect(sock, ep, use_awaitable);
  co_await async_write(sock, buf, use_awaitable);
  co_await async_read(sock, buf, use_awaitable);

```

| Approach | Readability | Error handling | Performance |
| --- | --- | --- | --- |
| Callbacks | Poor (nested) | Error codes in each callback | Best |
| Futures | Medium | exception from .get() | Overhead |
| Coroutines | Best (sequential) | try/catch around co_await | Excellent |

---

## Self-Assessment

### Q1: Async echo client with co_await

```cpp

// Requires: Asio (standalone or Boost) + C++20 compiler
// Compile: g++ -std=c++20 -fcoroutines -I/path/to/asio -lpthread client.cpp
#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::ip::tcp;
using asio::use_awaitable;

asio::awaitable<void> echo_client(asio::io_context& io) {
    // Resolve the server address
    tcp::resolver resolver(io);
    auto endpoints = co_await resolver.async_resolve(
        "127.0.0.1", "8080", use_awaitable);

    // Connect
    tcp::socket socket(io);
    co_await asio::async_connect(socket, endpoints, use_awaitable);
    std::cout << "Connected to server\n";

    // Send a message
    std::string message = "Hello from coroutine client!\n";
    co_await asio::async_write(
        socket, asio::buffer(message), use_awaitable);
    std::cout << "Sent: " << message;

    // Read the echo response
    char reply[1024];
    size_t n = co_await socket.async_read_some(
        asio::buffer(reply), use_awaitable);
    std::cout << "Received: " << std::string(reply, n) << '\n';

    // Socket closed automatically (RAII) when coroutine exits
}

int main() {
    asio::io_context io;

    // co_spawn launches the coroutine on the io_context
    asio::co_spawn(io, echo_client(io), [](std::exception_ptr ep) {
        if (ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << '\n';
            }
        }
    });

    io.run();  // drives the event loop
}
// Test: first start an echo server on port 8080, then run this client

```

### Q2: Error handling with try/catch around co_await

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>
#include <chrono>

using asio::ip::tcp;
using asio::use_awaitable;

asio::awaitable<void> robust_client(asio::io_context& io) {
    tcp::socket socket(io);

    // Error handling: each co_await can throw system_error
    try {
        // Connect with timeout using parallel awaiting
        tcp::resolver resolver(io);
        auto endpoints = co_await resolver.async_resolve(
            "127.0.0.1", "8080", use_awaitable);

        co_await asio::async_connect(socket, endpoints, use_awaitable);
        std::cout << "Connected!\n";

    } catch (const asio::system_error& e) {
        std::cerr << "Connection failed: " << e.what() << '\n';
        co_return;  // exit coroutine cleanly
    }

    // Read/write with error handling
    try {
        for (int i = 0; i < 3; ++i) {
            std::string msg = "Message " + std::to_string(i) + "\n";
            co_await asio::async_write(
                socket, asio::buffer(msg), use_awaitable);

            char buf[1024];
            size_t n = co_await socket.async_read_some(
                asio::buffer(buf), use_awaitable);
            std::cout << "Reply: " << std::string(buf, n);
        }
    } catch (const asio::system_error& e) {
        if (e.code() == asio::error::eof) {
            std::cout << "Server closed connection\n";
        } else {
            std::cerr << "I/O error: " << e.what() << '\n';
        }
    }
    // socket destructor closes the connection (RAII)
}

int main() {
    asio::io_context io;
    asio::co_spawn(io, robust_client(io), asio::detached);
    io.run();
}

```

### Q3: co_spawn integrates coroutines into io_context

`asio::co_spawn` is the bridge between C++20 coroutines and Asio's event loop:

```cpp

asio::co_spawn(executor, coroutine, completion_token)
  |
  v

1. Creates the coroutine frame (heap allocation)
2. Registers it with the executor (io_context)
3. Resumes the coroutine when async ops complete
4. Calls completion_token when coroutine finishes/throws

```

```cpp

#include <asio.hpp>
#include <asio/co_spawn.hpp>
#include <asio/use_awaitable.hpp>
#include <iostream>

using asio::use_awaitable;

asio::awaitable<int> compute() {
    // This coroutine runs on the io_context's thread
    asio::steady_timer timer(co_await asio::this_coro::executor,
                             std::chrono::seconds(1));
    co_await timer.async_wait(use_awaitable);
    co_return 42;
}

int main() {
    asio::io_context io;

    // Option 1: detached (fire and forget)
    asio::co_spawn(io, compute(), asio::detached);

    // Option 2: callback on completion
    asio::co_spawn(io, compute(), [](std::exception_ptr ep, int result) {
        if (ep) {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << '\n';
            }
        } else {
            std::cout << "Result: " << result << '\n';  // 42
        }
    });

    // Option 3: use_future (C++20)
    // auto fut = asio::co_spawn(io, compute(), asio::use_future);
    // io.run();
    // std::cout << fut.get() << '\n';

    io.run();  // MUST call run() to drive the event loop!
    // Without run(): nothing happens (coroutine is suspended, never resumed)
}

```

**Key points about co_spawn:**

- The coroutine starts executing immediately until the first `co_await`.
- After `co_await`, the coroutine is suspended and control returns to `io.run()`.
- When the async operation completes, `io.run()` resumes the coroutine.
- Multiple coroutines can be co_spawned — they run concurrently on the same thread.
- `asio::detached` ignores the result; use a callback to capture it.

---

## Notes

- Complementary to #729 (echo server + thread pool) and #550 (echo server + timeouts).
- `use_awaitable` makes async ops return awaitables (co_await-compatible).
- Coroutine frame allocation: Asio uses allocator-awareness to reduce heap allocs.
- For cancellation: use `asio::cancellation_signal` + `asio::bind_cancellation_slot`.
- GCC: needs `-fcoroutines`; Clang: needs `-stdlib=libc++`; MSVC: works by default.
