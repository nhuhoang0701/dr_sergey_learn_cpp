# Use AI for architecture design discussions and trade-off analysis

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI assistants serve as valuable **design discussion partners** for C++ architecture. They can evaluate multiple design options, analyze trade-offs (performance vs. maintainability, simplicity vs. flexibility), suggest patterns, and identify potential issues. While AI cannot replace experienced architects, it accelerates the exploration phase and helps surface considerations you might miss.

### AI Architecture Assistance Areas

| Design Activity | AI Strength | Best Prompt Strategy |
| --- | --- | --- |
| Compare design patterns | Excellent | "Compare X vs Y for this scenario" |
| Identify trade-offs | Good | Provide constraints + requirements |
| Suggest abstractions | Good | Describe the problem domain clearly |
| API surface design | Good | Describe consumers + use cases |
| Data structure selection | Excellent | Describe access patterns + sizes |
| Concurrency architecture | Medium | Describe threading model + invariants |
| Performance vs. clarity | Good | Specify latency/throughput targets |

---

## Self-Assessment

### Q1: Use AI for design option analysis

**Answer:**

```cpp

=== PROMPT ===

"I'm designing a plugin system for a C++ application.
Requirements:

- Plugins are loaded at runtime from shared libraries
- Each plugin provides 1+ 'processors' for a data pipeline
- Plugins must be isolated (crash in one doesn't kill the host)
- Hot-reload: update a plugin without restarting the app
- Performance: each processor handles 100K events/sec
- Must work on Linux and Windows

Compare these approaches:

1. dlopen/LoadLibrary with C interface
2. Shared memory + separate processes per plugin
3. gRPC-based out-of-process plugins
4. WASM sandbox (e.g., Wasmtime)

For each, analyze: performance overhead, crash isolation,
hot-reload feasibility, complexity, and cross-platform story."


=== EXPECTED AI RESPONSE STRUCTURE ===

"Option 1: dlopen/LoadLibrary with C API

+ Performance: minimal overhead, direct function calls
+ Hot-reload: dlclose + dlopen works
- Isolation: NONE - plugin crash kills host
- Cross-platform: need #ifdef for dl vs LoadLibrary

Complexity: Low

Option 2: Shared memory + separate processes

+ Isolation: full process isolation
+ Hot-reload: restart child process
- Performance: IPC overhead (shared mem is fast though)
- Complexity: HIGH (shared memory management, sync)
- Cross-platform: different IPC APIs

Complexity: High

Option 3: gRPC out-of-process

+ Isolation: full process isolation
+ Cross-platform: excellent (gRPC handles it)
+ Hot-reload: restart plugin process
- Performance: serialization overhead (~10-50μs per call)
- Dependency: heavy (gRPC + protobuf)

Complexity: Medium-High

Option 4: WASM sandbox

+ Isolation: WASM sandbox is memory-safe
+ Cross-platform: excellent
- Performance: ~2-10x overhead vs native
- C++ ecosystem: limited, toolchain complexity
- Hot-reload: easy (reload WASM module)

Complexity: Medium

RECOMMENDATION for your requirements:
Option 1 + watchdog for 100K events/sec with partial
isolation. Full isolation: Option 2 with lock-free
shared memory queues."

```

```cpp

// The AI might suggest a hybrid approach:

// Plugin interface (C ABI for dlopen compatibility)
extern "C" {
    struct PluginAPI {
        int version;
        const char* (*name)();
        void* (*create_processor)(const char* config_json);
        int (*process)(void* ctx, const Event* events, size_t count);
        void (*destroy_processor)(void* ctx);
    };
    PluginAPI* get_plugin_api();
}

// Host-side plugin manager with hot-reload
class PluginManager {
    struct LoadedPlugin {
        void* handle;          // dlopen handle
        PluginAPI* api;
        fs::file_time_type last_modified;
    };
    std::unordered_map<std::string, LoadedPlugin> plugins_;

public:
    void reload_if_changed(const std::string& path) {
        auto& p = plugins_[path];
        auto mod_time = fs::last_write_time(path);
        if (mod_time > p.last_modified) {
            // Drain pipeline, unload old, load new
            if (p.handle) dlclose(p.handle);
            p.handle = dlopen(path.c_str(), RTLD_NOW);
            p.api = reinterpret_cast<PluginAPI*(*)()>(
                dlsym(p.handle, "get_plugin_api"))();
            p.last_modified = mod_time;
        }
    }
};

```

### Q2: Trade-off analysis for data structure choices

**Answer:**

```cpp

=== PROMPT ===

"I need a concurrent data structure for a real-time trading
system. Requirements:

- Multiple producer threads insert orders (100K/sec)
- Single consumer thread processes in price-priority order
- Need fast cancel-by-ID (latency-critical)
- Memory: up to 1M live orders
- Latency budget: <1μs for insert, <500ns for cancel

Compare:

1. std::priority_queue + mutex
2. Lock-free skip list
3. Sorted array with binary search
4. Custom B-tree with fine-grained locking
5. Per-price-level linked lists (LOB style)

Analyze: insert/cancel/top complexity, cache behavior,
and implementation complexity."

```

```cpp

// AI's likely recommendation: Option 5 (Limit Order Book style)
// Reason: O(1) top, O(1) insert at known price, O(1) cancel with ID map

class OrderBook {
    // Price level: doubly-linked list of orders at same price
    struct PriceLevel {
        double price;
        std::list<Order> orders;  // O(1) insert/remove
    };

    // Sorted price levels (few distinct prices)
    std::map<double, PriceLevel, std::greater<>> bids_;  // Best bid first
    std::map<double, PriceLevel, std::less<>> asks_;      // Best ask first

    // Fast cancel: order ID -> iterator into the list
    std::unordered_map<uint64_t, std::list<Order>::iterator> order_index_;

public:
    void insert(Order order) {
        auto& level = bids_[order.price];
        level.orders.push_back(std::move(order));
        order_index_[order.id] = std::prev(level.orders.end());
    }

    bool cancel(uint64_t order_id) {  // O(1) amortized
        auto it = order_index_.find(order_id);
        if (it == order_index_.end()) return false;
        // Remove from price level list
        auto& level = bids_[it->second->price];
        level.orders.erase(it->second);
        order_index_.erase(it);
        return true;
    }

    const Order& best_bid() const {  // O(1)
        return bids_.begin()->second.orders.front();
    }
};

// Trade-off summary (AI output):
// | Operation | PQ+mutex | Skip list | LOB style |
// |-----------|----------|-----------|-----------|
// | Insert    | O(log N) | O(log N)  | O(log P)  |  P = price levels << N
// | Cancel    | O(N)     | O(log N)  | O(1)      |
// | Best      | O(1)     | O(1)      | O(1)      |
// | Concurrency| Poor    | Good      | Medium    |

```

### Q3: Use AI for API design review

**Answer:**

```text

=== PROMPT ===

"Review this C++ API design for a connection pool.
Identify design issues, suggest improvements, and
evaluate the trade-offs of each suggestion:

class ConnectionPool {
public:
    ConnectionPool(string host, int port, int maxConns);
    Connection* getConnection();  // Returns raw pointer
    void releaseConnection(Connection* conn);
    void closeAll();
    int activeCount();
    int idleCount();
};

Consider: RAII, thread safety, error handling,
testability, and modern C++ practices."

=== AI REVIEW ===

```

```cpp

// Issue 1: Raw pointer return - caller might forget to release
// Issue 2: No RAII - if exception thrown between get/release, leak
// Issue 3: No error handling for exhausted pool
// Issue 4: Not thread-safe (no mention of synchronization)
// Issue 5: Hardcoded Connection type - not testable

// AI-suggested improved API:

template<typename ConnectionT = Connection>
class ConnectionPool {
public:
    struct Config {
        std::string host;
        uint16_t port;
        size_t max_connections = 10;
        std::chrono::milliseconds acquire_timeout{5000};
        std::chrono::milliseconds idle_timeout{60000};
    };

    explicit ConnectionPool(Config config);

    // RAII handle - auto-returns to pool on destruction
    class PooledConnection {
        ConnectionT* conn_;
        ConnectionPool* pool_;
    public:
        ConnectionT& operator*()  { return *conn_; }
        ConnectionT* operator->() { return conn_; }
        ~PooledConnection() { if (pool_) pool_->release(conn_); }
        // Move-only
        PooledConnection(PooledConnection&&) noexcept;
        PooledConnection& operator=(PooledConnection&&) noexcept;
    };

    // Returns RAII handle, blocks until available or timeout
    [[nodiscard]]
    std::expected<PooledConnection, PoolError> acquire();

    // Try acquire without blocking
    std::optional<PooledConnection> try_acquire();

    // Stats (thread-safe)
    struct Stats {
        size_t active;
        size_t idle;
        size_t total_acquired;
        size_t total_timeouts;
    };
    Stats stats() const;

    // Graceful shutdown
    void drain(std::chrono::milliseconds timeout = std::chrono::seconds{30});
};

// Usage: RAII, exception-safe
{
    auto conn = pool.acquire();
    if (conn) {
        conn->execute("SELECT ...");
    }  // Auto-returned to pool here
}

```

---

## Notes

- **Provide concrete constraints** in prompts (latency targets, memory limits, throughput) not just "make it fast"
- AI is most useful in the **exploration phase** — narrowing options before detailed design
- Always **challenge AI's recommendations** — ask "what's the downside?" and "when would this fail?"
- For data structure selection, include the **access pattern** (read-heavy vs write-heavy, hot path operations)
- Ask AI to generate **comparison tables** for design options — very effective format
- AI tends to suggest **over-engineered solutions** — explicitly say "keep it simple" if needed
- Use AI for **API review before publishing** — it catches usability issues from a fresh perspective
