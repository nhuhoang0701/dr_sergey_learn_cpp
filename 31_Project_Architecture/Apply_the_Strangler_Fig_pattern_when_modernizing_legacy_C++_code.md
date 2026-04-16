# Apply the Strangler Fig pattern when modernizing legacy C++ code

**Category:** Project Architecture

---

## Topic Overview

The **Strangler Fig pattern** incrementally replaces a legacy system by routing new functionality through a modern implementation while keeping the old system running. Named after strangler fig trees that grow around a host tree, it avoids risky "big bang" rewrites. In C++ this applies to migrating from C-style code to modern C++, replacing monoliths with modular architectures, or upgrading from C++98 to C++20.

### Migration Strategies Compared

| Strategy | Risk | Duration | Downtime | Legacy Runs? |
| --- | --- | --- | --- | --- |
| **Big bang rewrite** | Very high | Long | Extended | No (replaced) |
| **Strangler Fig** | Low | Gradual | Zero | Yes (shrinks) |
| **Branch by abstraction** | Medium | Medium | Zero | Partially |
| **Feature toggle** | Low | Gradual | Zero | Both run |

---

## Self-Assessment

### Q1: Implement a facade to route between legacy and modern implementations

**Answer:**

```cpp

#include <memory>
#include <string>
#include <unordered_set>
#include <functional>
#include <iostream>

// === Legacy system (C-style, global state) ===
namespace legacy {
    struct order_t {
        int id;
        char customer[64];
        double total;
        int status;  // 0=pending, 1=shipped, 2=cancelled
    };

    int create_order(const char* customer, double total);
    int cancel_order(int order_id);
    order_t* get_order(int id);  // Returns raw pointer to static storage
}

// === Modern replacement ===
namespace modern {
    class Order {
    public:
        enum class Status { Pending, Shipped, Cancelled };
        std::string id;
        std::string customer;
        double total;
        Status status = Status::Pending;
    };

    class OrderService {
    public:
        std::string create(const std::string& customer, double total);
        void cancel(const std::string& id);
        std::optional<Order> find(const std::string& id);
    };
}

// === Strangler facade: routes to legacy or modern ===
class OrderFacade {
public:
    OrderFacade(modern::OrderService& modern_svc)
        : modern_(modern_svc) {}

    // Route based on feature completion
    std::string create_order(const std::string& customer, double total) {
        if (use_modern("create_order")) {
            return modern_.create(customer, total);
        }
        // Legacy path (adapter)
        int id = legacy::create_order(customer.c_str(), total);
        return std::to_string(id);
    }

    void cancel_order(const std::string& id) {
        if (use_modern("cancel_order")) {
            modern_.cancel(id);
        } else {
            legacy::cancel_order(std::stoi(id));
        }
    }

    // Gradually migrate features
    void enable_modern(const std::string& feature) {
        modern_features_.insert(feature);
        std::cout << "Migrated to modern: " << feature << "\n";
    }

private:
    bool use_modern(const std::string& feature) const {
        return modern_features_.count(feature) > 0;
    }

    modern::OrderService& modern_;
    std::unordered_set<std::string> modern_features_;
};

// === Gradual migration timeline ===
// Week 1: facade.enable_modern("create_order");
// Week 3: facade.enable_modern("cancel_order");
// Week 6: facade.enable_modern("get_order");
// Week 8: Remove legacy code entirely

```

### Q2: Implement data migration with dual-write

**Answer:**

```cpp

// === Dual-write: both systems receive writes during migration ===
class DualWriteOrderService {
public:
    DualWriteOrderService(legacy::Database& old_db,
                          modern::Database& new_db)
        : old_db_(old_db), new_db_(new_db) {}

    std::string create_order(const OrderRequest& req) {
        // Write to legacy (primary)
        int legacy_id = old_db_.insert_order(req);

        // Write to modern (secondary, best-effort during migration)
        try {
            auto modern_id = new_db_.insert_order(req, legacy_id);
            id_mapping_[legacy_id] = modern_id;
        } catch (const std::exception& e) {
            log_migration_error("create", legacy_id, e.what());
            // Don't fail the operation; legacy is still primary
        }

        return std::to_string(legacy_id);
    }

    // Phase 2: Modern becomes primary, legacy is secondary
    std::string create_order_v2(const OrderRequest& req) {
        auto modern_id = new_db_.insert_order(req);

        // Write to legacy (secondary, for rollback safety)
        try {
            old_db_.insert_order(req);
        } catch (...) {
            // Legacy write failure is non-fatal
        }

        return modern_id;
    }

    // Phase 3: Modern only (legacy removed)
    std::string create_order_v3(const OrderRequest& req) {
        return new_db_.insert_order(req);
    }

private:
    legacy::Database& old_db_;
    modern::Database& new_db_;
    std::unordered_map<int, std::string> id_mapping_;
};

// === Data comparison tool: verify migration correctness ===
class MigrationVerifier {
public:
    struct Mismatch {
        std::string field;
        std::string legacy_value;
        std::string modern_value;
    };

    std::vector<Mismatch> compare(int legacy_id,
                                  const std::string& modern_id) {
        std::vector<Mismatch> mismatches;

        auto legacy_order = legacy_db_.get_order(legacy_id);
        auto modern_order = modern_db_.get_order(modern_id);

        if (legacy_order.total != modern_order.total) {
            mismatches.push_back({"total",
                std::to_string(legacy_order.total),
                std::to_string(modern_order.total)});
        }
        // ... compare other fields
        return mismatches;
    }

    void verify_all() {
        int mismatched = 0;
        for (auto& [old_id, new_id] : id_mapping_) {
            auto issues = compare(old_id, new_id);
            if (!issues.empty()) {
                ++mismatched;
                log_mismatches(old_id, issues);
            }
        }
        std::cout << "Verification: "
                  << id_mapping_.size() - mismatched
                  << "/" << id_mapping_.size()
                  << " records match\n";
    }
};

```

### Q3: Branch by abstraction for internal module replacement

**Answer:**

```cpp

// === Branch by abstraction: introduce interface, swap implementation ===

// Step 1: Extract interface from legacy code
class IPaymentProcessor {
public:
    virtual ~IPaymentProcessor() = default;
    virtual PaymentResult charge(const std::string& account,
                                  double amount) = 0;
    virtual PaymentResult refund(const std::string& transaction_id) = 0;
};

// Step 2: Wrap legacy behind the interface
class LegacyPaymentAdapter : public IPaymentProcessor {
public:
    PaymentResult charge(const std::string& account, double amount) override {
        // Legacy C-style call
        int result = legacy_charge_account(account.c_str(),
                                           (int)(amount * 100));
        return result == 0 ? PaymentResult::Success
                           : PaymentResult::Failed;
    }

    PaymentResult refund(const std::string& txn_id) override {
        return legacy_refund(std::stoi(txn_id)) == 0
            ? PaymentResult::Success
            : PaymentResult::Failed;
    }
};

// Step 3: Build modern implementation behind same interface
class ModernPaymentProcessor : public IPaymentProcessor {
public:
    PaymentResult charge(const std::string& account, double amount) override {
        auto txn = gateway_.create_transaction(account, amount);
        return txn.execute();
    }

    PaymentResult refund(const std::string& txn_id) override {
        return gateway_.refund(txn_id);
    }

private:
    PaymentGateway gateway_;
};

// Step 4: Toggle between implementations
class PaymentProcessorFactory {
public:
    static std::unique_ptr<IPaymentProcessor> create(Config& config) {
        if (config.get_bool("payments.use_modern")) {
            return std::make_unique<ModernPaymentProcessor>();
        }
        return std::make_unique<LegacyPaymentAdapter>();
    }
};

// Step 5: Shadow mode — run both, compare results
class ShadowPaymentProcessor : public IPaymentProcessor {
public:
    ShadowPaymentProcessor(
        std::unique_ptr<IPaymentProcessor> primary,
        std::unique_ptr<IPaymentProcessor> shadow)
        : primary_(std::move(primary)), shadow_(std::move(shadow)) {}

    PaymentResult charge(const std::string& account, double amount) override {
        auto primary_result = primary_->charge(account, amount);

        // Shadow execution: compare but don't affect result
        try {
            auto shadow_result = shadow_->charge(account, amount);
            if (shadow_result != primary_result) {
                log_shadow_mismatch("charge", primary_result,
                                     shadow_result);
            }
        } catch (...) {
            log_shadow_error("charge");
        }

        return primary_result;  // Always return primary's result
    }

private:
    std::unique_ptr<IPaymentProcessor> primary_;
    std::unique_ptr<IPaymentProcessor> shadow_;
};

```

---

## Notes

- **Never do a big bang rewrite** of a working system — Strangler Fig is always safer
- The facade is the key abstraction: all consumers use it, unaware of routing
- **Dual-write** ensures data parity during migration; verify with comparison tools
- **Shadow mode** runs both implementations side-by-side and compares results before switching
- Feature flags control the route: percentage rollouts (5% modern → 50% → 100%)
- Each migration phase should be independently deployable and rollback-safe
- Keep legacy tests green throughout — they validate that the facade preserves behavior
