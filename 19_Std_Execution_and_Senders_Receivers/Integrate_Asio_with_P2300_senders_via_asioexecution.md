# Integrate Asio with P2300 senders via asio::execution

**Category:** std::execution & Senders/Receivers  
**Item:** #612  
**Standard:** C++20  
**Reference:** <https://think-async.com/Asio/>  

---

## Topic Overview

Boost.Asio and P2300 (std::execution) are complementary: Asio provides async I/O, while P2300 provides composable async pipelines. Bridging them lets you use Asio's io_context as a P2300 scheduler.

| Concept | Asio | P2300 |
| --- | --- | --- |
| Where work runs | `io_context` / `strand` | scheduler / execution context |
| Async operation | `async_read`, `async_write` | sender |
| Callback model | Completion handler | receiver |
| Composition | `async_compose` | `then`, `when_all`, pipe `\|` |

---

## Self-Assessment

### Q1: Wrap Asio `async_read_some` into a P2300 sender

```cpp

// asio_sender.cpp — wrapping Asio async_read into a sender
#include <stdexec/execution.hpp>
#include <boost/asio.hpp>
#include <iostream>
#include <vector>
#include <system_error>

namespace asio = boost::asio;
using tcp = asio::ip::tcp;

// Sender that wraps async_read_some:
struct asio_read_sender {
    using sender_concept = stdexec::sender_t;
    using completion_signatures = stdexec::completion_signatures<
        stdexec::set_value_t(std::size_t),             // bytes read
        stdexec::set_error_t(std::exception_ptr),
        stdexec::set_stopped_t()
    >;

    tcp::socket& socket;
    asio::mutable_buffer buffer;

    template <typename Receiver>
    struct operation {
        using operation_state_concept = stdexec::operation_state_t;
        tcp::socket& socket;
        asio::mutable_buffer buffer;
        Receiver receiver;

        void start() noexcept {
            // Bridge: Asio completion handler -> P2300 receiver
            socket.async_read_some(
                buffer,
                [this](std::error_code ec, std::size_t bytes_read) {
                    if (ec == asio::error::operation_aborted) {
                        stdexec::set_stopped(std::move(receiver));
                    } else if (ec) {
                        stdexec::set_error(std::move(receiver),
                            std::make_exception_ptr(
                                std::system_error(ec)));
                    } else {
                        stdexec::set_value(std::move(receiver),
                                          bytes_read);
                    }
                });
        }
    };

    template <stdexec::receiver Receiver>
    auto connect(Receiver recv) const noexcept {
        return operation<Receiver>{socket, buffer, std::move(recv)};
    }
};

auto async_read_some(tcp::socket& sock, asio::mutable_buffer buf) {
    return asio_read_sender{sock, buf};
}

// Usage in a sender pipeline:
void example(tcp::socket& sock) {
    std::vector<char> buf(1024);
    auto pipeline = async_read_some(sock, asio::buffer(buf))
        | stdexec::then([&buf](std::size_t n) {
            std::string data(buf.data(), n);
            std::cout << "Read: " << data << '\n';
            return n;
        })
        | stdexec::upon_error([](std::exception_ptr ep) -> std::size_t {
            try { std::rethrow_exception(ep); }
            catch (const std::exception& e) {
                std::cerr << "Error: " << e.what() << '\n';
            }
            return 0;
        });
    // Drive with io_context::run() instead of sync_wait
}

```

### Q2: Asio `io_context` as a P2300 scheduler

```cpp

// asio_scheduler.cpp — adapt io_context to P2300 scheduler concept
#include <stdexec/execution.hpp>
#include <boost/asio.hpp>
#include <iostream>
#include <thread>

namespace asio = boost::asio;

// Adapter: wraps io_context as a P2300 scheduler
struct asio_scheduler {
    asio::io_context& ctx;

    // schedule() returns a sender that completes on the io_context
    struct schedule_sender {
        using sender_concept = stdexec::sender_t;
        using completion_signatures = stdexec::completion_signatures<
            stdexec::set_value_t()
        >;
        asio::io_context& ctx;

        template <typename Receiver>
        struct op {
            using operation_state_concept = stdexec::operation_state_t;
            asio::io_context& ctx;
            Receiver recv;

            void start() noexcept {
                // Post work to the io_context:
                asio::post(ctx, [this]() {
                    stdexec::set_value(std::move(recv));
                });
            }
        };

        template <stdexec::receiver Receiver>
        auto connect(Receiver recv) const noexcept {
            return op<Receiver>{ctx, std::move(recv)};
        }
    };

    auto schedule() const noexcept {
        return schedule_sender{ctx};
    }

    bool operator==(const asio_scheduler&) const = default;
};

int main() {
    asio::io_context io;
    asio_scheduler sched{io};

    // Use our Asio scheduler in a P2300 pipeline:
    auto work = sched.schedule()
        | stdexec::then([]() {
            std::cout << "Running on io_context thread: "
                      << std::this_thread::get_id() << '\n';
            return 42;
        });

    // Run io_context in another thread to demonstrate scheduling:
    std::thread io_thread([&io]() { io.run(); });

    // Note: sync_wait won't work here since io_context drives the work.
    // In practice, integrate with io.run() event loop.
    io_thread.join();
}

```

### Q3: Compatibility between Asio executors and P2300 schedulers

| Asio Executor Model | P2300 Scheduler Model |
| --- | --- |
| `executor.post(fn)` | `schedule(sched) \| then(fn)` |
| `executor.dispatch(fn)` | `schedule(sched) \| then(fn)` (inline if on context) |
| `strand<executor>` | No direct equivalent (use mutex or serializing scheduler) |
| `io_context::run()` | `sync_wait()` or event loop on main thread |
| `any_io_executor` | Type-erased scheduler |

Bridging strategy:

```cpp

Asio world:                         P2300 world:
┌──────────────────┐              ┌──────────────────┐
│  io_context      │─── adapt ──→│ P2300 scheduler   │
│  async_read()    │─── wrap  ──→│ sender            │
│  completion hdlr │─── map   ──→│ receiver signals  │
└──────────────────┘              └──────────────────┘

Key mapping:
  ec == operation_aborted  →  set_stopped()
  ec (other error)         →  set_error(exception_ptr)
  success                  →  set_value(result)

```

---

## Notes

- Asio's `deferred` completion token returns sender-like objects natively.
- Asio 1.30+ has experimental support for P2300-compatible senders via `asio::experimental::use_sender`.
- The main challenge is event loop ownership: Asio uses `io_context::run()`, P2300 uses `sync_wait()`.
- Strands don't map directly to P2300 — but you can create a serializing sender adapter.
- Consider `asio::co_spawn` for coroutine-based bridging as an alternative.
