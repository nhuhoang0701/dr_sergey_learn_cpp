# Design event-driven architectures with message queues in C++

**Category:** Project Architecture

---

## Topic Overview

**Event-driven architecture (EDA)** decouples producers from consumers through asynchronous event dispatch. Components publish events without knowing who handles them; subscribers react independently. Combined with **message queues**, EDA enables loose coupling, scalability, and fault tolerance. In C++ this ranges from in-process event buses to inter-process queues (ZeroMQ, RabbitMQ, Kafka).

### Event-Driven vs Request-Response

| Aspect | Request-Response | Event-Driven |
| --- | --- | --- |
| Coupling | Caller knows callee | Publisher blind to consumers |
| Flow | Synchronous | Asynchronous |
| Scaling | Bounded by callee | Independent scaling |
| Ordering | Request order | Event ordering guarantees needed |
| Failure | Cascading | Isolated (queue buffers) |

---

## Self-Assessment

### Q1: Implement a type-safe in-process event bus

**Answer:**

```cpp

#include <functional>
#include <unordered_map>
#include <typeindex>
#include <vector>
#include <memory>
#include <any>
#include <mutex>

// === Type-safe Event Bus ===
class EventBus {
public:
    // Subscribe to events of type E
    template<typename E>
    size_t subscribe(std::function<void(const E&)> handler) {
        std::lock_guard lock(mutex_);
        auto id = next_id_++;
        auto wrapper = [handler](const std::any& e) {
            handler(std::any_cast<const E&>(e));
        };
        handlers_[typeid(E)].push_back({id, std::move(wrapper)});
        return id;
    }

    // Unsubscribe by ID
    void unsubscribe(size_t id) {
        std::lock_guard lock(mutex_);
        for (auto& [type, entries] : handlers_) {
            entries.erase(
                std::remove_if(entries.begin(), entries.end(),
                    [id](const Entry& e) { return e.id == id; }),
                entries.end());
        }
    }

    // Publish event to all subscribers
    template<typename E>
    void publish(const E& event) {
        std::vector<std::function<void(const std::any&)>> to_call;
        {
            std::lock_guard lock(mutex_);
            auto it = handlers_.find(typeid(E));
            if (it != handlers_.end())
                for (auto& entry : it->second)
                    to_call.push_back(entry.handler);
        }
        // Call outside lock to avoid deadlocks
        for (auto& fn : to_call)
            fn(std::any(event));
    }

private:
    struct Entry {
        size_t id;
        std::function<void(const std::any&)> handler;
    };
    std::unordered_map<std::type_index, std::vector<Entry>> handlers_;
    std::mutex mutex_;
    size_t next_id_ = 1;
};

// === Events ===
struct OrderPlaced {
    int order_id;
    double total;
    std::string customer;
};

struct PaymentProcessed {
    int order_id;
    bool success;
};

struct OrderShipped {
    int order_id;
    std::string tracking;
};

// === Subscribing ===
void setup(EventBus& bus) {
    // Payment service reacts to orders
    bus.subscribe<OrderPlaced>([&bus](const OrderPlaced& e) {
        bool ok = process_payment(e.total);
        bus.publish(PaymentProcessed{e.order_id, ok});
    });

    // Shipping reacts to successful payments
    bus.subscribe<PaymentProcessed>([&bus](const PaymentProcessed& e) {
        if (e.success) {
            auto tracking = ship_order(e.order_id);
            bus.publish(OrderShipped{e.order_id, tracking});
        }
    });

    // Notification reacts to shipping
    bus.subscribe<OrderShipped>([](const OrderShipped& e) {
        send_email("Your order " + std::to_string(e.order_id)

                   + " shipped: " + e.tracking);

    });
}

```

### Q2: Add an async message queue with worker threads

**Answer:**

```cpp

#include <queue>
#include <thread>
#include <condition_variable>
#include <atomic>

// === Thread-safe message queue ===
template<typename T>
class MessageQueue {
public:
    void push(T msg) {
        {
            std::lock_guard lock(mutex_);
            queue_.push(std::move(msg));
        }
        cv_.notify_one();
    }

    T pop() {
        std::unique_lock lock(mutex_);
        cv_.wait(lock, [this] { return !queue_.empty() || stopped_; });
        if (stopped_ && queue_.empty())
            throw std::runtime_error("Queue stopped");
        T msg = std::move(queue_.front());
        queue_.pop();
        return msg;
    }

    bool try_pop(T& msg, std::chrono::milliseconds timeout) {
        std::unique_lock lock(mutex_);
        if (!cv_.wait_for(lock, timeout,
                [this] { return !queue_.empty() || stopped_; }))
            return false;
        if (stopped_ && queue_.empty()) return false;
        msg = std::move(queue_.front());
        queue_.pop();
        return true;
    }

    void stop() {
        {
            std::lock_guard lock(mutex_);
            stopped_ = true;
        }
        cv_.notify_all();
    }

    size_t size() const {
        std::lock_guard lock(mutex_);
        return queue_.size();
    }

private:
    std::queue<T> queue_;
    mutable std::mutex mutex_;
    std::condition_variable cv_;
    bool stopped_ = false;
};

// === Async Event Processor with worker pool ===
class AsyncEventProcessor {
public:
    using Task = std::function<void()>;

    explicit AsyncEventProcessor(int workers = 4)
        : workers_(workers) {
        for (int i = 0; i < workers; ++i) {
            threads_.emplace_back([this] { worker_loop(); });
        }
    }

    template<typename E>
    void enqueue(E event, std::function<void(const E&)> handler) {
        queue_.push([event = std::move(event),
                     handler = std::move(handler)]() {
            handler(event);
        });
    }

    ~AsyncEventProcessor() {
        queue_.stop();
        for (auto& t : threads_) t.join();
    }

private:
    void worker_loop() {
        while (true) {
            try {
                auto task = queue_.pop();
                task();
            } catch (...) {
                break;  // Queue stopped
            }
        }
    }

    MessageQueue<Task> queue_;
    std::vector<std::jthread> threads_;
    int workers_;
};

```

### Q3: Implement event sourcing with replay

**Answer:**

```cpp

// === Event Store: persist all events for replay ===
class EventStore {
public:
    struct StoredEvent {
        size_t sequence;
        std::string type;
        std::any data;
        std::chrono::system_clock::time_point timestamp;
    };

    template<typename E>
    void append(const E& event) {
        events_.push_back({
            next_seq_++,
            typeid(E).name(),
            std::any(event),
            std::chrono::system_clock::now()
        });
    }

    // Replay all events through handler
    template<typename E>
    void replay(std::function<void(const E&)> handler) const {
        for (const auto& evt : events_) {
            if (evt.type == typeid(E).name()) {
                handler(std::any_cast<const E&>(evt.data));
            }
        }
    }

    // Replay from a specific sequence number
    std::vector<StoredEvent> since(size_t seq) const {
        std::vector<StoredEvent> result;
        for (const auto& evt : events_)
            if (evt.sequence > seq)
                result.push_back(evt);
        return result;
    }

    size_t latest_sequence() const {
        return events_.empty() ? 0 : events_.back().sequence;
    }

private:
    std::vector<StoredEvent> events_;
    size_t next_seq_ = 1;
};

// === Rebuild state from events ===
class OrderAggregate {
public:
    void apply(const OrderPlaced& e) {
        id_ = e.order_id;
        total_ = e.total;
        status_ = "placed";
    }
    void apply(const PaymentProcessed& e) {
        status_ = e.success ? "paid" : "payment_failed";
    }
    void apply(const OrderShipped& e) {
        tracking_ = e.tracking;
        status_ = "shipped";
    }

    // Rebuild from event store:
    // EventStore store;
    // OrderAggregate order;
    // store.replay<OrderPlaced>([&](auto& e) { order.apply(e); });
    // store.replay<PaymentProcessed>([&](auto& e) { order.apply(e); });

    std::string status() const { return status_; }

private:
    int id_ = 0;
    double total_ = 0;
    std::string status_;
    std::string tracking_;
};

```

---

## Notes

- **Event bus** for in-process: simple, fast, same-process communication
- **Message queue** for cross-thread or cross-process: buffered, back-pressure capable
- Always copy subscribers list before invoking — handlers may modify the subscriber list
- Event ordering matters: use sequence numbers and single-writer per aggregate
- Event sourcing gives full audit trail and state rebuild capability but increases storage
- For high-throughput, consider lock-free SPSC (single-producer, single-consumer) queues
- Real-world queues: ZeroMQ (in-process/IPC/TCP), Redis Streams, Apache Kafka for distributed
