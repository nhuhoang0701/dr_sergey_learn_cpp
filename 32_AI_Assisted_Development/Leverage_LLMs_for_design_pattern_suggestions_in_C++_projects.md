# Leverage LLMs for design pattern suggestions in C++ projects

**Category:** AI-Assisted C++ Development

---

## Topic Overview

LLMs can analyze code structure and suggest appropriate **design patterns**, explain when to apply them, generate the implementation, and identify when a pattern is being misused. For C++, AI is especially useful because modern C++ patterns differ significantly from classic GoF patterns — AI can suggest **modern alternatives** using templates, concepts, `std::variant`, and CRTP.

### Design Pattern AI Assistance

| Task | AI Effectiveness | Example Prompt |
| --- | --- | --- |
| Identify applicable patterns | Good | "What pattern fits this problem?" |
| Implement a pattern in modern C++ | Excellent | "Implement Observer using C++20" |
| Refactor to a pattern | Good | "Refactor these switches to Visitor" |
| Choose between patterns | Good | "Strategy vs Policy: when to use which?" |
| Modernize classic patterns | Excellent | "Replace virtual Observer with std::function" |
| Anti-pattern detection | Medium | "Is this a God class? How to refactor?" |

---

## Self-Assessment

### Q1: Get pattern suggestions from AI

**Answer:**

```cpp

=== PROMPT ===

"I have this code structure. Suggest which design pattern
would improve it and show the modern C++ implementation.

Problem: Multiple message types are processed differently.
Currently using a chain of if/else with dynamic_cast.
New message types require modifying the processor class.
Need to add new types frequently without changing existing code."

```

```cpp

// CURRENT CODE (problematic):
void MessageProcessor::process(Message* msg) {
    if (auto* text = dynamic_cast<TextMessage*>(msg)) {
        log_text(text->content());
        store_text(text);
    } else if (auto* image = dynamic_cast<ImageMessage*>(msg)) {
        validate_image(image);
        compress_image(image);
        store_image(image);
    } else if (auto* video = dynamic_cast<VideoMessage*>(msg)) {
        transcode_video(video);
        store_video(video);
    }
    // Adding new type = modify this function!
}


// === AI SUGGESTION: std::variant + Visitor (Modern C++17) ===

// Define message types as value types (no inheritance)
struct TextMessage {
    std::string content;
    std::string sender;
};

struct ImageMessage {
    std::vector<std::byte> data;
    int width, height;
};

struct VideoMessage {
    std::filesystem::path path;
    std::chrono::seconds duration;
};

// Variant holds any message type
using Message = std::variant<TextMessage, ImageMessage, VideoMessage>;

// Visitors for different operations
struct LogVisitor {
    void operator()(const TextMessage& m) const {
        spdlog::info("Text from {}: {}", m.sender, m.content);
    }
    void operator()(const ImageMessage& m) const {
        spdlog::info("Image {}x{}", m.width, m.height);
    }
    void operator()(const VideoMessage& m) const {
        spdlog::info("Video {}", m.path.string());
    }
};

struct ProcessVisitor {
    void operator()(const TextMessage& m) const {
        store_text(m);
    }
    void operator()(ImageMessage& m) const {
        compress(m);
        store_image(m);
    }
    void operator()(VideoMessage& m) const {
        transcode(m);
        store_video(m);
    }
};

// Usage: adding new type = add to variant + add handler in visitors
// Compiler enforces exhaustive handling!
void process(Message& msg) {
    std::visit(LogVisitor{}, msg);
    std::visit(ProcessVisitor{}, msg);
}

```

### Q2: Compare pattern alternatives with AI

**Answer:**

```cpp

=== PROMPT ===

"Compare these three approaches for a notification system
in C++. 100K subscribers, real-time delivery, subscribers
added/removed frequently:

1. Classic Observer (virtual base class)
2. Signal/Slot (std::function + vector)
3. Event bus with type-erased messages

Show the trade-offs table and recommend one."

```

```cpp

// === Option 1: Classic Observer ===
class IObserver {
public:
    virtual ~IObserver() = default;
    virtual void on_event(const Event& e) = 0;
};

class Subject {
    std::vector<IObserver*> observers_;  // Raw pointers: lifetime issue!
public:
    void subscribe(IObserver* o) { observers_.push_back(o); }
    void notify(const Event& e) {
        for (auto* o : observers_) o->on_event(e);
    }
};
// Pros: Simple, well-known
// Cons: Lifetime management, virtual call overhead, tight coupling


// === Option 2: Signal/Slot (AI-recommended for this case) ===
template<typename... Args>
class Signal {
    struct Slot {
        uint64_t id;
        std::function<void(Args...)> callback;
    };
    std::vector<Slot> slots_;
    uint64_t next_id_ = 0;

public:
    // Returns connection ID for unsubscribe
    [[nodiscard]] uint64_t connect(std::function<void(Args...)> fn) {
        uint64_t id = next_id_++;
        slots_.push_back({id, std::move(fn)});
        return id;
    }

    void disconnect(uint64_t id) {
        std::erase_if(slots_,
            [id](const Slot& s) { return s.id == id; });
    }

    void emit(Args... args) const {
        for (const auto& slot : slots_) {
            slot.callback(args...);
        }
    }
};

// Usage:
Signal<const Order&> order_created;

auto id = order_created.connect([](const Order& o) {
    send_email(o.customer_email());
});

order_created.emit(new_order);
order_created.disconnect(id);
// Pros: Decoupled, type-safe, no base class needed
// Cons: std::function overhead, not thread-safe as shown


// === Option 3: Event Bus ===
class EventBus {
    using Handler = std::function<void(const void*)>;
    std::unordered_map<std::type_index, std::vector<Handler>> handlers_;

public:
    template<typename E>
    void subscribe(std::function<void(const E&)> handler) {
        handlers_[typeid(E)].push_back(
            [h = std::move(handler)](const void* e) {
                h(*static_cast<const E*>(e));
            });
    }

    template<typename E>
    void publish(const E& event) {
        auto it = handlers_.find(typeid(E));
        if (it != handlers_.end()) {
            for (auto& handler : it->second)
                handler(&event);
        }
    }
};
// Pros: Fully decoupled, any event type
// Cons: Type erasure overhead, no compile-time checks

// AI RECOMMENDATION:
// "For 100K subscribers with frequent add/remove:
//  Signal/Slot with a slot_map (O(1) disconnect).
//  Use a stable vector to avoid iterator invalidation
//  during emit. Add mutex for thread safety."

```

### Q3: Ask AI to detect and fix anti-patterns

**Answer:**

```cpp

=== PROMPT ===

"Analyze this class for design problems.
Identify anti-patterns and suggest refactoring steps
with modern C++ patterns."

```

```cpp

// Code to analyze:
class AppManager {
    Database* db_;
    HttpClient* http_;
    Logger* logger_;
    ConfigManager* config_;
    CacheManager* cache_;
    EmailService* email_;
    MetricsCollector* metrics_;

public:
    void process_order(Order& order) {
        logger_->log("Processing order " + order.id());
        metrics_->increment("orders.processed");

        auto customer = db_->query("SELECT * FROM customers WHERE id=" + order.customer_id());
        if(!customer) { logger_->log("Customer not found"); return; }

        auto price = http_->get("/pricing/" + order.product_id());
        order.set_price(std::stod(price.body()));

        if (order.total() > 1000) {
            email_->send(config_->get("admin_email"), "Large order: " + order.id());
        }

        cache_->invalidate("customer:" + order.customer_id());
        db_->execute("INSERT INTO orders ...");
    }

    void process_refund(Order& order) { /* similar mess */ }
    void generate_report() { /* uses db_, http_, email_ */ }
    void sync_inventory() { /* uses db_, http_, cache_ */ }
    // ... 20 more methods using different combinations
};


// === AI ANALYSIS ===

// Anti-patterns detected:
// 1. GOD CLASS: AppManager does everything (ordering, refunds,
//    reports, inventory). Violates SRP.
// 2. RAW POINTERS: No clear ownership. Who deletes these?
// 3. SQL INJECTION: String concatenation in queries.
// 4. HIDDEN DEPENDENCIES: 7 dependencies injected, all coupled.

// AI-suggested refactoring:

// Step 1: Extract focused services
class OrderService {
    std::shared_ptr<IOrderRepository> repo_;
    std::shared_ptr<IPricingClient> pricing_;
    std::shared_ptr<INotificationService> notifications_;
public:
    std::expected<OrderResult, OrderError>
    process(const OrderRequest& request) {
        auto customer = repo_->find_customer(request.customer_id);
        if (!customer) return std::unexpected(OrderError::CustomerNotFound);

        auto price = pricing_->get_price(request.product_id);
        if (!price) return std::unexpected(OrderError::PricingUnavailable);

        Order order{request, *customer, *price};

        if (auto saved = repo_->save(order); !saved)
            return std::unexpected(OrderError::SaveFailed);

        if (order.total() > 1000.0)
            notifications_->notify_large_order(order);

        return OrderResult{order.id()};
    }
};

// Step 2: Use interfaces for testability
class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;
    virtual std::optional<Customer> find_customer(std::string_view id) = 0;
    virtual bool save(const Order& order) = 0;
};

```

---

## Notes

- AI suggests **modern C++ patterns** (variant + visit, CRTP, concepts) over classic inheritance-heavy GoF
- Ask AI to compare **virtual dispatch vs CRTP vs std::variant** for your specific use case
- For **God class** refactoring, ask AI to "identify the responsibilities and split into focused classes"
- AI can generate **pattern implementation skeletons** that you fill with domain logic
- Always specify **performance requirements** — AI chooses differently for low-latency vs. flexibility
- Ask AI to identify when a pattern is **overkill** — "Is Observer worth it for 3 subscribers?"
- Use AI to **translate between languages**: "show me the C++ equivalent of this Rust trait pattern"
