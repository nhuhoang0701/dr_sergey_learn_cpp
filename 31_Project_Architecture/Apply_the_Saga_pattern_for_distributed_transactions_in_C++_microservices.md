# Apply the Saga pattern for distributed transactions in C++ microservices

**Category:** Project Architecture

---

## Topic Overview

The **Saga pattern** manages distributed transactions across multiple services without a global lock. Each step executes a local transaction and publishes an event. If any step fails, **compensating transactions** undo the effects of previous steps in reverse order. There are two coordination approaches: **choreography** (event-driven, decentralized) and **orchestration** (centralized controller).

### Choreography vs Orchestration

| Aspect | Choreography | Orchestration |
| --- | --- | --- |
| Coordination | Each service listens and reacts | Central orchestrator manages flow |
| Coupling | Looser (event-driven) | Services coupled to orchestrator |
| Complexity | Hard to trace full flow | Clear, centralized flow |
| Error handling | Distributed, harder | Centralized, easier |
| Best for | Simple sagas (2-3 steps) | Complex sagas (4+ steps) |

---

## Self-Assessment

### Q1: Implement orchestration-based saga

**Answer:**

```cpp

#include <string>
#include <vector>
#include <functional>
#include <optional>
#include <variant>
#include <iostream>

// === Saga step: action + compensating action ===
struct SagaStep {
    std::string name;
    std::function<bool()> execute;      // Returns true on success
    std::function<void()> compensate;   // Undo on failure
};

// === Saga orchestrator ===
class SagaOrchestrator {
public:
    enum class Status { NotStarted, Running, Completed, Compensating, Failed };

    void add_step(SagaStep step) {
        steps_.push_back(std::move(step));
    }

    Status execute() {
        status_ = Status::Running;
        completed_steps_ = 0;

        for (size_t i = 0; i < steps_.size(); ++i) {
            std::cout << "Executing step: " << steps_[i].name << "\n";

            if (!steps_[i].execute()) {
                std::cout << "Step failed: " << steps_[i].name
                          << ", compensating...\n";
                compensate(i);
                status_ = Status::Failed;
                return status_;
            }
            completed_steps_ = i + 1;
        }

        status_ = Status::Completed;
        return status_;
    }

private:
    void compensate(size_t failed_step) {
        status_ = Status::Compensating;
        // Compensate in reverse order, EXCLUDING the failed step
        for (int i = static_cast<int>(failed_step) - 1; i >= 0; --i) {
            std::cout << "Compensating step: " << steps_[i].name << "\n";
            try {
                steps_[i].compensate();
            } catch (const std::exception& e) {
                // Log but continue compensating remaining steps
                std::cerr << "Compensation failed for "
                          << steps_[i].name << ": " << e.what() << "\n";
            }
        }
    }

    std::vector<SagaStep> steps_;
    Status status_ = Status::NotStarted;
    size_t completed_steps_ = 0;
};

// === Order creation saga ===
SagaOrchestrator create_order_saga(
    OrderService& orders, PaymentService& payments,
    InventoryService& inventory, ShippingService& shipping,
    const OrderRequest& request) {

    SagaOrchestrator saga;
    std::string order_id, payment_id, reservation_id;

    saga.add_step({
        "Create Order",
        [&]() {
            order_id = orders.create(request);
            return !order_id.empty();
        },
        [&]() { orders.cancel(order_id); }
    });

    saga.add_step({
        "Reserve Inventory",
        [&]() {
            reservation_id = inventory.reserve(request.items);
            return !reservation_id.empty();
        },
        [&]() { inventory.release(reservation_id); }
    });

    saga.add_step({
        "Process Payment",
        [&]() {
            payment_id = payments.charge(request.customer_id,
                                         request.total);
            return !payment_id.empty();
        },
        [&]() { payments.refund(payment_id); }
    });

    saga.add_step({
        "Schedule Shipping",
        [&]() {
            return shipping.schedule(order_id, request.address);
        },
        [&]() { shipping.cancel(order_id); }
    });

    return saga;
}

// Usage:
// auto saga = create_order_saga(orders, payments, inventory, shipping, req);
// auto status = saga.execute();
// If payment fails -> inventory released, order cancelled (reverse order)

```

### Q2: Implement choreography-based saga with events

**Answer:**

```cpp

// === Choreography: each service reacts to events ===

// Events
struct OrderCreated { std::string order_id; OrderDetails details; };
struct InventoryReserved { std::string order_id; std::string reservation_id; };
struct InventoryFailed { std::string order_id; std::string reason; };
struct PaymentProcessed { std::string order_id; std::string payment_id; };
struct PaymentFailed { std::string order_id; std::string reason; };
struct OrderCompleted { std::string order_id; };
struct OrderCancelled { std::string order_id; std::string reason; };

// Inventory service listens for OrderCreated
class InventoryService {
public:
    explicit InventoryService(EventBus& bus) : bus_(bus) {
        bus.subscribe<OrderCreated>([this](const OrderCreated& e) {
            try {
                auto res_id = reserve(e.details.items);
                bus_.publish(InventoryReserved{e.order_id, res_id});
            } catch (...) {
                bus_.publish(InventoryFailed{e.order_id, "Out of stock"});
            }
        });

        // Compensate if payment fails
        bus.subscribe<PaymentFailed>([this](const PaymentFailed& e) {
            release_for_order(e.order_id);
        });
    }
    // ...
private:
    EventBus& bus_;
};

// Payment service listens for InventoryReserved
class PaymentService {
public:
    explicit PaymentService(EventBus& bus) : bus_(bus) {
        bus.subscribe<InventoryReserved>([this](const InventoryReserved& e) {
            try {
                auto pay_id = charge(e.order_id);
                bus_.publish(PaymentProcessed{e.order_id, pay_id});
            } catch (...) {
                bus_.publish(PaymentFailed{e.order_id, "Insufficient funds"});
            }
        });
    }
private:
    EventBus& bus_;
};

// Order service listens for completion/failure
class OrderService {
public:
    explicit OrderService(EventBus& bus) : bus_(bus) {
        bus.subscribe<PaymentProcessed>([this](const PaymentProcessed& e) {
            complete_order(e.order_id);
            bus_.publish(OrderCompleted{e.order_id});
        });

        bus.subscribe<InventoryFailed>([this](const InventoryFailed& e) {
            cancel_order(e.order_id, e.reason);
        });

        bus.subscribe<PaymentFailed>([this](const PaymentFailed& e) {
            cancel_order(e.order_id, e.reason);
        });
    }
private:
    EventBus& bus_;
};

```

### Q3: Saga with timeout, retry, and idempotency

**Answer:**

```cpp

// === Production-grade saga step with retry and timeout ===
struct ResilientSagaStep {
    std::string name;
    std::function<bool()> execute;
    std::function<void()> compensate;
    int max_retries = 3;
    std::chrono::milliseconds timeout{5000};
    std::chrono::milliseconds retry_delay{1000};
};

class ResilientSagaOrchestrator {
public:
    void add_step(ResilientSagaStep step) {
        steps_.push_back(std::move(step));
    }

    bool execute() {
        for (size_t i = 0; i < steps_.size(); ++i) {
            if (!execute_with_retry(steps_[i])) {
                compensate(i);
                return false;
            }
        }
        return true;
    }

private:
    bool execute_with_retry(const ResilientSagaStep& step) {
        for (int attempt = 0; attempt <= step.max_retries; ++attempt) {
            if (attempt > 0) {
                std::cout << "Retry " << attempt << "/"
                          << step.max_retries << ": "
                          << step.name << "\n";
                std::this_thread::sleep_for(step.retry_delay * attempt);
            }

            // Execute with timeout
            auto future = std::async(std::launch::async, step.execute);
            if (future.wait_for(step.timeout) == std::future_status::ready) {
                if (future.get()) return true;  // Success
            } else {
                std::cerr << "Timeout: " << step.name << "\n";
            }
        }
        return false;  // All retries exhausted
    }

    void compensate(size_t failed_step) {
        for (int i = static_cast<int>(failed_step) - 1; i >= 0; --i) {
            // Compensations also retry
            for (int attempt = 0; attempt <= 3; ++attempt) {
                try {
                    steps_[i].compensate();
                    break;  // Success
                } catch (...) {
                    if (attempt == 3) {
                        // Dead letter: manual intervention needed
                        log_dead_letter(steps_[i].name);
                    }
                }
            }
        }
    }

    std::vector<ResilientSagaStep> steps_;
};

// === Idempotency: prevent duplicate execution ===
class IdempotencyStore {
public:
    // Returns true if this is a new operation
    bool try_acquire(const std::string& idempotency_key) {
        std::lock_guard lock(mutex_);
        return completed_.insert(idempotency_key).second;
    }

    void release(const std::string& key) {
        std::lock_guard lock(mutex_);
        completed_.erase(key);
    }

private:
    std::mutex mutex_;
    std::unordered_set<std::string> completed_;
};

// Wrap step with idempotency
auto make_idempotent(IdempotencyStore& store, const std::string& key,
                     std::function<bool()> action) {
    return [&store, key, action]() -> bool {
        if (!store.try_acquire(key)) {
            return true;  // Already executed, consider success
        }
        return action();
    };
}

```

---

## Notes

- Use **orchestration** for complex workflows; **choreography** for simple 2-3 step flows
- Compensating transactions must be **idempotent** — they may be called multiple times
- Compensations that fail need a **dead letter queue** for manual resolution
- Each saga step should be atomic at the local service level
- Saga state must be persisted to survive crashes (store step index + status in DB)
- Timeout + retry is essential for cross-service calls; use exponential backoff
- Sagas provide **eventual consistency**, not ACID — design UIs to handle intermediate states
