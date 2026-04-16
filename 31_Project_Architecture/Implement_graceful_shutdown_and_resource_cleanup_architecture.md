# Implement graceful shutdown and resource cleanup architecture

**Category:** Project Architecture

---

## Topic Overview

**Graceful shutdown** ensures all resources are released, pending work completes (or is safely canceled), and data integrity is preserved when an application terminates. In C++ this requires handling OS signals, ordering destruction of subsystems, draining work queues, and flushing persistent state. A poorly designed shutdown causes data corruption, resource leaks, and hung processes.

### Shutdown Challenges

| Challenge | Risk | Solution |
| --- | --- | --- |
| Signal handling | UB in signal handlers | `sig_atomic_t` flag + event loop check |
| Thread joining | Deadlock on shutdown | Stop tokens, poison messages |
| Resource ordering | Use-after-free | Reverse initialization order |
| Pending I/O | Data loss | Drain queues before stop |
| Static destruction | Order undefined | Explicit cleanup before `main` returns |

---

## Self-Assessment

### Q1: Implement signal-safe graceful shutdown

**Answer:**

```cpp

#include <csignal>
#include <atomic>
#include <thread>
#include <iostream>
#include <vector>
#include <functional>

// === Signal-safe shutdown flag ===
std::atomic<bool> g_shutdown_requested{false};

// Signal handler: only set atomic flag (async-signal-safe)
void signal_handler(int signum) {
    g_shutdown_requested.store(true, std::memory_order_relaxed);
}

// === Shutdown coordinator ===
class ShutdownCoordinator {
public:
    // Register cleanup functions (called in reverse order)
    void on_shutdown(std::string name, std::function<void()> cleanup) {
        cleanups_.push_back({std::move(name), std::move(cleanup)});
    }

    void install_signal_handlers() {
        struct sigaction sa{};
        sa.sa_handler = signal_handler;
        sigemptyset(&sa.sa_mask);
        sa.sa_flags = 0;
        sigaction(SIGINT, &sa, nullptr);
        sigaction(SIGTERM, &sa, nullptr);
    }

    bool shutdown_requested() const {
        return g_shutdown_requested.load(std::memory_order_relaxed);
    }

    void execute_shutdown() {
        std::cout << "Shutdown initiated...\n";
        // Execute in reverse registration order (LIFO)
        for (auto it = cleanups_.rbegin(); it != cleanups_.rend(); ++it) {
            std::cout << "  Cleaning up: " << it->name << "\n";
            try {
                it->cleanup();
            } catch (const std::exception& e) {
                std::cerr << "  Error during " << it->name
                          << ": " << e.what() << "\n";
            }
        }
        std::cout << "Shutdown complete.\n";
    }

private:
    struct Entry {
        std::string name;
        std::function<void()> cleanup;
    };
    std::vector<Entry> cleanups_;
};

```

### Q2: Implement subsystem lifecycle with ordered shutdown

**Answer:**

```cpp

// === Subsystem interface ===
class ISubsystem {
public:
    virtual ~ISubsystem() = default;
    virtual const char* name() const = 0;
    virtual void start() = 0;
    virtual void request_stop() = 0;  // Non-blocking: signal to stop
    virtual void wait_stopped() = 0;  // Blocking: wait for completion
};

// === Application with ordered lifecycle ===
class Application {
public:
    void add_subsystem(std::unique_ptr<ISubsystem> sub) {
        subsystems_.push_back(std::move(sub));
    }

    void start() {
        coordinator_.install_signal_handlers();

        // Start in order
        for (auto& sub : subsystems_) {
            std::cout << "Starting: " << sub->name() << "\n";
            sub->start();
        }
    }

    void run() {
        // Main loop: wait for shutdown signal
        while (!coordinator_.shutdown_requested()) {
            std::this_thread::sleep_for(std::chrono::milliseconds(100));
        }
    }

    void shutdown() {
        // Phase 1: Request all to stop (non-blocking)
        for (auto it = subsystems_.rbegin(); it != subsystems_.rend(); ++it) {
            std::cout << "Requesting stop: " << (*it)->name() << "\n";
            (*it)->request_stop();
        }

        // Phase 2: Wait for each to finish (blocking, reverse order)
        for (auto it = subsystems_.rbegin(); it != subsystems_.rend(); ++it) {
            std::cout << "Waiting: " << (*it)->name() << "\n";
            (*it)->wait_stopped();
            std::cout << "Stopped: " << (*it)->name() << "\n";
        }
    }

private:
    ShutdownCoordinator coordinator_;
    std::vector<std::unique_ptr<ISubsystem>> subsystems_;
};

// === Example subsystem: worker threads ===
class WorkerPool : public ISubsystem {
public:
    const char* name() const override { return "WorkerPool"; }

    void start() override {
        for (int i = 0; i < 4; ++i) {
            workers_.emplace_back([this] {
                while (!stop_.load()) {
                    // Process work...
                    std::this_thread::sleep_for(
                        std::chrono::milliseconds(10));
                }
            });
        }
    }

    void request_stop() override {
        stop_.store(true);
    }

    void wait_stopped() override {
        for (auto& w : workers_) w.join();
        workers_.clear();
    }

private:
    std::vector<std::jthread> workers_;
    std::atomic<bool> stop_{false};
};

// === main ===
int main() {
    Application app;
    app.add_subsystem(std::make_unique<DatabaseSubsystem>());
    app.add_subsystem(std::make_unique<WorkerPool>());
    app.add_subsystem(std::make_unique<HttpServer>());

    app.start();    // DB -> Workers -> HTTP
    app.run();      // Wait for SIGINT/SIGTERM
    app.shutdown(); // HTTP -> Workers -> DB (reverse order)
}

```

### Q3: Handle shutdown with C++20 stop_token

**Answer:**

```cpp

#include <stop_token>
#include <thread>

// === jthread with stop_token for cooperative cancellation ===
class ModernWorkerPool {
public:
    void start() {
        for (int i = 0; i < 4; ++i) {
            // jthread passes stop_token automatically
            workers_.emplace_back([this](std::stop_token st) {
                while (!st.stop_requested()) {
                    // Check stop_token in inner loops too
                    process_batch(st);
                }
            });
        }
    }

    void request_stop() {
        // jthread::request_stop() signals all workers
        for (auto& w : workers_)
            w.request_stop();
    }

    void wait_stopped() {
        workers_.clear();  // jthread destructor joins
    }

private:
    void process_batch(std::stop_token st) {
        for (int i = 0; i < 1000 && !st.stop_requested(); ++i) {
            // Process item...
        }
    }

    std::vector<std::jthread> workers_;
};

// === Drain a message queue on shutdown ===
template<typename T>
class DrainableQueue {
public:
    void push(T item) {
        std::lock_guard lock(mutex_);
        if (closed_) return;  // Reject new items during shutdown
        queue_.push(std::move(item));
        cv_.notify_one();
    }

    // Close the queue: no new items, drain existing
    void close() {
        std::lock_guard lock(mutex_);
        closed_ = true;
        cv_.notify_all();
    }

    // Returns nullopt when queue is closed AND empty
    std::optional<T> pop() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || closed_; });
        if (queue_.empty()) return std::nullopt;
        T item = std::move(queue_.front());
        queue_.pop();
        return item;
    }

    size_t size() const {
        std::lock_guard lock(mutex_);
        return queue_.size();
    }

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool closed_ = false;
};

```

---

## Notes

- **Signal handlers must be async-signal-safe**: only set an atomic flag, nothing else
- Shutdown order = reverse of startup: stops dependents before their dependencies
- Two-phase shutdown: `request_stop()` (fast, non-blocking) then `wait_stopped()` (blocking)
- C++20 `std::stop_token` / `std::jthread` simplifies cooperative cancellation
- **Drain queues** before stopping workers — otherwise pending work is lost
- Reject new work during shutdown: mark queues as closed
- Avoid `std::atexit` for complex cleanup — explicit shutdown in main() is more controllable
