# Implement actor model concurrency with message passing in C++

**Category:** Project Architecture

---

## Topic Overview

The **Actor Model** treats concurrent entities as isolated actors that communicate exclusively through asynchronous message passing. Each actor has a mailbox (message queue), processes messages sequentially, and can create child actors or send messages to other actors. This eliminates shared mutable state and the need for locks entirely.

### Actor Model vs Shared-State Concurrency

| Aspect | Actor Model | Shared State + Locks |
| --- | --- | --- |
| Data sharing | None (message copies) | Shared memory + mutexes |
| Deadlocks | Impossible (no locks) | Common (lock ordering) |
| Data races | Impossible | Require careful locking |
| Debugging | Trace messages | Reproduce timing |
| Overhead | Message copies, queues | Lock contention |
| Best for | I/O-heavy, distributed | CPU-intensive, low-latency |

---

## Self-Assessment

### Q1: Implement a basic actor framework

**Answer:**

```cpp

#include <any>
#include <functional>
#include <memory>
#include <queue>
#include <mutex>
#include <condition_variable>
#include <thread>
#include <unordered_map>

class Actor;
using ActorRef = std::shared_ptr<Actor>;

// === Message ===
struct Message {
    ActorRef sender;
    std::any payload;
};

// === Actor base class ===
class Actor : public std::enable_shared_from_this<Actor> {
public:
    virtual ~Actor() { stop(); }

    // Send message to this actor
    void send(Message msg) {
        {
            std::lock_guard lock(mutex_);
            mailbox_.push(std::move(msg));
        }
        cv_.notify_one();
    }

    // Convenience: send typed payload
    template<typename T>
    void tell(const T& payload, ActorRef sender = nullptr) {
        send({sender, std::any(payload)});
    }

    void start() {
        running_ = true;
        thread_ = std::jthread([this] { run_loop(); });
    }

    void stop() {
        {
            std::lock_guard lock(mutex_);
            running_ = false;
        }
        cv_.notify_all();
        if (thread_.joinable()) thread_.join();
    }

protected:
    // Override to handle messages
    virtual void on_receive(const Message& msg) = 0;

private:
    void run_loop() {
        while (true) {
            Message msg;
            {
                std::unique_lock lock(mutex_);
                cv_.wait(lock, [this] {
                    return !mailbox_.empty() || !running_;
                });
                if (!running_ && mailbox_.empty()) break;
                msg = std::move(mailbox_.front());
                mailbox_.pop();
            }
            on_receive(msg);  // Sequential processing = no races
        }
    }

    std::queue<Message> mailbox_;
    std::mutex mutex_;
    std::condition_variable cv_;
    std::jthread thread_;
    bool running_ = false;
};

// === Actor System: manages actor lifecycle ===
class ActorSystem {
public:
    template<typename ActorType, typename... Args>
    ActorRef spawn(Args&&... args) {
        auto actor = std::make_shared<ActorType>(std::forward<Args>(args)...);
        actor->start();
        actors_.push_back(actor);
        return actor;
    }

    void shutdown() {
        for (auto& a : actors_) a->stop();
        actors_.clear();
    }

private:
    std::vector<ActorRef> actors_;
};

```

### Q2: Build concrete actors that communicate via messages

**Answer:**

```cpp

// === Message types ===
struct ProcessOrder { int order_id; double amount; };
struct OrderProcessed { int order_id; bool success; };
struct LogMessage { std::string text; };

// === Logger Actor ===
class LoggerActor : public Actor {
protected:
    void on_receive(const Message& msg) override {
        if (auto* log = std::any_cast<LogMessage>(&msg.payload)) {
            std::cout << "[LOG] " << log->text << "\n";
        }
    }
};

// === Payment Actor ===
class PaymentActor : public Actor {
public:
    explicit PaymentActor(ActorRef logger) : logger_(logger) {}

protected:
    void on_receive(const Message& msg) override {
        if (auto* order = std::any_cast<ProcessOrder>(&msg.payload)) {
            logger_->tell(LogMessage{
                "Processing payment for order " +
                std::to_string(order->order_id)});

            bool success = order->amount < 10000;  // Business rule

            // Reply to sender
            if (msg.sender) {
                msg.sender->tell(
                    OrderProcessed{order->order_id, success},
                    shared_from_this());
            }
        }
    }

private:
    ActorRef logger_;
};

// === Order Actor: orchestrates ===
class OrderActor : public Actor {
public:
    OrderActor(ActorRef payment, ActorRef logger)
        : payment_(payment), logger_(logger) {}

protected:
    void on_receive(const Message& msg) override {
        if (auto* order = std::any_cast<ProcessOrder>(&msg.payload)) {
            logger_->tell(LogMessage{
                "New order " + std::to_string(order->order_id)});
            payment_->tell(*order, shared_from_this());
        }
        else if (auto* result = std::any_cast<OrderProcessed>(&msg.payload)) {
            auto status = result->success ? "succeeded" : "failed";
            logger_->tell(LogMessage{
                "Order " + std::to_string(result->order_id)

                + " " + status});

        }
    }

private:
    ActorRef payment_;
    ActorRef logger_;
};

// === Usage ===
int main() {
    ActorSystem system;
    auto logger  = system.spawn<LoggerActor>();
    auto payment = system.spawn<PaymentActor>(logger);
    auto orders  = system.spawn<OrderActor>(payment, logger);

    orders->tell(ProcessOrder{1, 99.99});
    orders->tell(ProcessOrder{2, 50000.00});

    std::this_thread::sleep_for(std::chrono::milliseconds(100));
    system.shutdown();
}

```

### Q3: Implement supervision and error recovery

**Answer:**

```cpp

// === Supervised actor: parent monitors children ===
class SupervisedActor : public Actor {
public:
    enum class Strategy { Restart, Stop, Escalate };

protected:
    void on_receive(const Message& msg) override {
        try {
            handle(msg);
        } catch (const std::exception& e) {
            auto strategy = decide(e);
            switch (strategy) {
                case Strategy::Restart:
                    on_restart();
                    break;
                case Strategy::Stop:
                    stop();
                    break;
                case Strategy::Escalate:
                    if (parent_)
                        parent_->tell(
                            ChildFailed{shared_from_this(), e.what()});
                    break;
            }
        }
    }

    virtual void handle(const Message& msg) = 0;
    virtual Strategy decide(const std::exception& e) {
        return Strategy::Restart;  // Default: restart
    }
    virtual void on_restart() {
        // Reset internal state, keep mailbox
    }

    ActorRef parent_;
};

struct ChildFailed {
    ActorRef child;
    std::string error;
};

// === Supervisor actor ===
class Supervisor : public Actor {
protected:
    void on_receive(const Message& msg) override {
        if (auto* fail = std::any_cast<ChildFailed>(&msg.payload)) {
            std::cout << "Child failed: " << fail->error << "\n";
            // Restart the child
            fail->child->stop();
            fail->child->start();
        }
    }
};

```

---

## Notes

- **Each actor processes messages sequentially** — no internal locking needed
- Actors communicate only through messages; never share pointers to internal state
- Message copies are the price of safety — use move semantics to minimize cost
- Supervision hierarchies (parent restarts children) provide fault tolerance
- For high-performance, use lock-free MPSC queues instead of `std::mutex` + `std::queue`
- Real-world C++ actor libraries: CAF (C++ Actor Framework), SObjectizer, rotor
- Actors scale naturally to distributed systems: replace local send with network send
