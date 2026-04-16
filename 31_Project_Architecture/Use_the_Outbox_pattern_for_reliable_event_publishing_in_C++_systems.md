# Use the Outbox pattern for reliable event publishing in C++ systems

**Category:** Project Architecture

---

## Topic Overview

The **Outbox pattern** solves the dual-write problem: when a service must update its database AND publish an event, either operation can fail independently, leading to inconsistency. The solution: write events to an "outbox" table in the same database transaction as the business data, then a separate process reads the outbox and publishes events to the message broker. This guarantees at-least-once delivery.

### Dual-Write Problem

| Scenario | DB Write | Event Publish | Result |
| --- | --- | --- | --- |
| Happy path | ✅ | ✅ | Consistent |
| Event fails | ✅ | ❌ | DB updated, no event — **inconsistent** |
| DB fails | ❌ | ✅ | Event sent, no data — **inconsistent** |
| Both fail | ❌ | ❌ | Consistent (nothing happened) |
| **Outbox** | ✅ (data + event in one txn) | Async relay | **Always consistent** |

---

## Self-Assessment

### Q1: Implement the outbox pattern with a database transaction

**Answer:**

```cpp

#include <string>
#include <vector>
#include <chrono>
#include <optional>

// === Outbox entry ===
struct OutboxEntry {
    int64_t id;
    std::string aggregate_type;  // e.g., "Order"
    std::string aggregate_id;    // e.g., order ID
    std::string event_type;      // e.g., "OrderCreated"
    std::string payload;         // JSON serialized event
    std::chrono::system_clock::time_point created_at;
    bool published = false;
};

// === Service writes business data + outbox in one transaction ===
class OrderService {
public:
    OrderService(Database& db) : db_(db) {}

    std::string create_order(const OrderRequest& req) {
        auto txn = db_.begin_transaction();

        try {
            // 1. Write business data
            Order order;
            order.id = generate_id();
            order.customer_id = req.customer_id;
            order.total = req.total;
            order.status = "pending";

            txn.execute(
                "INSERT INTO orders (id, customer_id, total, status) "
                "VALUES (?, ?, ?, ?)",
                order.id, order.customer_id, order.total, order.status);

            // 2. Write event to outbox (SAME transaction)
            std::string payload = serialize_event(
                OrderCreatedEvent{order.id, order.customer_id, order.total});

            txn.execute(
                "INSERT INTO outbox "
                "(aggregate_type, aggregate_id, event_type, payload) "
                "VALUES (?, ?, ?, ?)",
                "Order", order.id, "OrderCreated", payload);

            txn.commit();  // Both succeed or both fail
            return order.id;

        } catch (...) {
            txn.rollback();  // Both rolled back
            throw;
        }
    }

    void cancel_order(const std::string& order_id) {
        auto txn = db_.begin_transaction();

        txn.execute("UPDATE orders SET status = 'cancelled' WHERE id = ?",
                    order_id);

        txn.execute(
            "INSERT INTO outbox "
            "(aggregate_type, aggregate_id, event_type, payload) "
            "VALUES (?, ?, ?, ?)",
            "Order", order_id, "OrderCancelled",
            serialize_event(OrderCancelledEvent{order_id}));

        txn.commit();
    }

private:
    Database& db_;
};

```

### Q2: Implement the outbox relay (publisher)

**Answer:**

```cpp

// === Outbox relay: reads outbox and publishes to message broker ===
class OutboxRelay {
public:
    OutboxRelay(Database& db, MessageBroker& broker,
                std::chrono::milliseconds poll_interval = std::chrono::seconds(1))
        : db_(db), broker_(broker), poll_interval_(poll_interval) {}

    void start() {
        running_ = true;
        thread_ = std::jthread([this](std::stop_token st) {
            while (!st.stop_requested()) {
                process_batch();
                std::this_thread::sleep_for(poll_interval_);
            }
        });
    }

    void stop() {
        running_ = false;
        if (thread_.joinable()) thread_.request_stop();
    }

private:
    void process_batch() {
        // Fetch unpublished entries (with row locking)
        auto entries = db_.query<OutboxEntry>(
            "SELECT id, aggregate_type, aggregate_id, event_type, payload "
            "FROM outbox WHERE published = false "
            "ORDER BY id ASC LIMIT 100 FOR UPDATE SKIP LOCKED");

        for (const auto& entry : entries) {
            try {
                // Publish to message broker
                broker_.publish(
                    /*topic=*/entry.aggregate_type + "." + entry.event_type,
                    /*key=*/entry.aggregate_id,
                    /*payload=*/entry.payload,
                    /*headers=*/{{"event_id", std::to_string(entry.id)}}
                );

                // Mark as published
                db_.execute(
                    "UPDATE outbox SET published = true WHERE id = ?",
                    entry.id);

            } catch (const std::exception& e) {
                // Will retry on next poll (at-least-once)
                std::cerr << "Publish failed for outbox entry "
                          << entry.id << ": " << e.what() << "\n";
                break;  // Preserve ordering
            }
        }
    }

    Database& db_;
    MessageBroker& broker_;
    std::chrono::milliseconds poll_interval_;
    std::atomic<bool> running_{false};
    std::jthread thread_;
};

// === Outbox cleanup: purge old published entries ===
void cleanup_outbox(Database& db, std::chrono::hours retention) {
    auto cutoff = std::chrono::system_clock::now() - retention;
    db.execute(
        "DELETE FROM outbox WHERE published = true AND created_at < ?",
        cutoff);
}

```

### Q3: Idempotent consumer to handle duplicate events

**Answer:**

```cpp

// === Idempotent consumer: deduplicates at-least-once delivery ===
class IdempotentConsumer {
public:
    IdempotentConsumer(Database& db) : db_(db) {}

    // Returns true if this event was already processed
    bool is_duplicate(const std::string& event_id) {
        auto result = db_.query_scalar<int>(
            "SELECT COUNT(*) FROM processed_events WHERE event_id = ?",
            event_id);
        return result > 0;
    }

    // Process idempotently: skip if already done
    template<typename Event, typename Handler>
    void handle(const std::string& event_id, const Event& event,
                Handler&& handler) {
        if (is_duplicate(event_id)) {
            std::cout << "Skipping duplicate event: " << event_id << "\n";
            return;
        }

        auto txn = db_.begin_transaction();
        try {
            // Process the event
            handler(event, txn);

            // Record that we processed it (same transaction)
            txn.execute(
                "INSERT INTO processed_events (event_id, processed_at) "
                "VALUES (?, NOW())",
                event_id);

            txn.commit();
        } catch (...) {
            txn.rollback();
            throw;
        }
    }

private:
    Database& db_;
};

// === Consumer setup ===
void setup_consumer(MessageBroker& broker, IdempotentConsumer& consumer) {
    broker.subscribe("Order.OrderCreated",
        [&consumer](const Message& msg) {
            auto event_id = msg.header("event_id");
            auto event = deserialize<OrderCreatedEvent>(msg.payload());

            consumer.handle(event_id, event,
                [](const OrderCreatedEvent& e, Transaction& txn) {
                    // Process: e.g., send welcome email, update analytics
                    txn.execute(
                        "INSERT INTO order_analytics "
                        "(order_id, customer_id, amount) VALUES (?, ?, ?)",
                        e.order_id, e.customer_id, e.total);
                });
        });
}

// SQL schema:
// CREATE TABLE outbox (
//     id BIGSERIAL PRIMARY KEY,
//     aggregate_type VARCHAR(100) NOT NULL,
//     aggregate_id VARCHAR(100) NOT NULL,
//     event_type VARCHAR(100) NOT NULL,
//     payload JSONB NOT NULL,
//     published BOOLEAN DEFAULT false,
//     created_at TIMESTAMP DEFAULT NOW()
// );
// CREATE INDEX idx_outbox_unpublished ON outbox(published, id)
//     WHERE published = false;
//
// CREATE TABLE processed_events (
//     event_id VARCHAR(100) PRIMARY KEY,
//     processed_at TIMESTAMP DEFAULT NOW()
// );

```

---

## Notes

- The outbox guarantees **atomicity** between DB write and event: both in one transaction
- The relay provides **at-least-once delivery** — consumers must be idempotent
- **Polling** is simplest; **CDC (Change Data Capture)** via Debezium/WAL tailing is more efficient
- `FOR UPDATE SKIP LOCKED` allows multiple relay instances without conflicts
- Clean up old published entries periodically (e.g., retain 7 days)
- The outbox table acts as a persistent event log — useful for replay and debugging
- Alternative: use PostgreSQL LISTEN/NOTIFY instead of polling for lower latency
