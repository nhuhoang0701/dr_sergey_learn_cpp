# Design compile-time configuration using constexpr and templates

**Category:** Project Architecture

---

## Topic Overview

**Compile-time configuration** uses `constexpr`, `if constexpr`, template parameters, and preprocessor defines to select behavior at compile time rather than runtime. This eliminates dead code, enables zero-overhead abstractions, and catches misconfiguration as compile errors. It's especially valuable in embedded systems where every byte and cycle counts.

### Configuration Mechanisms

| Mechanism | Flexibility | Safety | Overhead |
| --- | --- | --- | --- |
| `constexpr` variables | Compile-time values | Type-safe | Zero |
| `if constexpr` | Branch elimination | Dead code removed | Zero |
| Template parameters | Type/value selection | Compile-time checked | Zero |
| `#define` | Preprocessor switching | Weak (text replacement) | Zero |
| Policy classes | Behavior injection | Strong (concepts) | Zero |

---

## Self-Assessment

### Q1: Design a compile-time configuration system

**Answer:**

```cpp

#include <cstdint>
#include <type_traits>

// === Configuration struct (all constexpr) ===
struct Config {
    static constexpr bool ENABLE_LOGGING = true;
    static constexpr bool ENABLE_ASSERTIONS = true;
    static constexpr int MAX_CONNECTIONS = 128;
    static constexpr size_t BUFFER_SIZE = 4096;
    static constexpr bool USE_TLS = true;
    static constexpr int LOG_LEVEL = 2;  // 0=off, 1=error, 2=info, 3=debug

    // Platform detection
    static constexpr bool IS_EMBEDDED =
#ifdef PLATFORM_EMBEDDED
        true;
#else
        false;
#endif

    // Derived configurations
    static constexpr size_t POOL_SIZE = IS_EMBEDDED ? 16 : 256;
    static constexpr bool USE_HEAP = !IS_EMBEDDED;
};

// === Zero-overhead logging ===
template<int Level>
void log_impl(const char* msg) {
    if constexpr (Config::ENABLE_LOGGING && Level <= Config::LOG_LEVEL) {
        // This entire function body is eliminated when logging is disabled
        if constexpr (Level == 1) {
            fprintf(stderr, "[ERROR] %s\n", msg);
        } else if constexpr (Level == 2) {
            printf("[INFO] %s\n", msg);
        } else if constexpr (Level == 3) {
            printf("[DEBUG] %s\n", msg);
        }
    }
    // When ENABLE_LOGGING is false, this function is empty = no overhead
}

#define LOG_ERROR(msg) log_impl<1>(msg)
#define LOG_INFO(msg)  log_impl<2>(msg)
#define LOG_DEBUG(msg) log_impl<3>(msg)

// === Compile-time assert for config validation ===
static_assert(Config::MAX_CONNECTIONS > 0,
    "MAX_CONNECTIONS must be positive");
static_assert(Config::BUFFER_SIZE >= 256,
    "BUFFER_SIZE too small");
static_assert(!Config::IS_EMBEDDED || Config::POOL_SIZE <= 64,
    "Pool too large for embedded");

```

### Q2: Use policy classes for compile-time behavior injection

**Answer:**

```cpp

// === Policy classes: inject behavior at compile time ===

// Allocation policy
struct HeapAllocation {
    template<typename T>
    static T* allocate(size_t n) {
        return new T[n];
    }
    template<typename T>
    static void deallocate(T* p) {
        delete[] p;
    }
};

struct PoolAllocation {
    template<typename T>
    static T* allocate(size_t n) {
        static std::array<std::byte, 4096> pool;
        static size_t offset = 0;
        T* ptr = reinterpret_cast<T*>(pool.data() + offset);
        offset += sizeof(T) * n;
        return ptr;
    }
    template<typename T>
    static void deallocate(T*) {
        // Pool doesn't free individual allocations
    }
};

// Threading policy
struct SingleThreaded {
    struct Lock { Lock() {} };
    // Zero-overhead: no mutex, no atomic
};

struct MultiThreaded {
    struct Lock {
        Lock() { mutex_.lock(); }
        ~Lock() { mutex_.unlock(); }
        static std::mutex mutex_;
    };
};

// === Class parameterized by policies ===
template<typename AllocPolicy = HeapAllocation,
         typename ThreadPolicy = MultiThreaded>
class ConnectionPool {
public:
    void add_connection() {
        typename ThreadPolicy::Lock lock;
        auto* conn = AllocPolicy::template allocate<Connection>(1);
        connections_.push_back(conn);
    }

    void remove_connection(Connection* conn) {
        typename ThreadPolicy::Lock lock;
        // ...
        AllocPolicy::template deallocate(conn);
    }

private:
    std::vector<Connection*> connections_;
};

// Compile-time configuration:
using ProdPool = ConnectionPool<HeapAllocation, MultiThreaded>;
using EmbeddedPool = ConnectionPool<PoolAllocation, SingleThreaded>;
using TestPool = ConnectionPool<HeapAllocation, SingleThreaded>;

```

### Q3: Compile-time feature flags with concepts

**Answer:**

```cpp

#include <concepts>

// === Feature flags as types ===
struct FeatureEnabled  { static constexpr bool value = true; };
struct FeatureDisabled { static constexpr bool value = false; };

// === Build configuration as a type bundle ===
struct ProductionConfig {
    using Logging   = FeatureEnabled;
    using Metrics   = FeatureEnabled;
    using Tracing   = FeatureDisabled;
    using DebugUI   = FeatureDisabled;
    static constexpr size_t WorkerCount = 16;
};

struct DebugConfig {
    using Logging   = FeatureEnabled;
    using Metrics   = FeatureEnabled;
    using Tracing   = FeatureEnabled;
    using DebugUI   = FeatureEnabled;
    static constexpr size_t WorkerCount = 2;
};

struct EmbeddedConfig {
    using Logging   = FeatureDisabled;
    using Metrics   = FeatureDisabled;
    using Tracing   = FeatureDisabled;
    using DebugUI   = FeatureDisabled;
    static constexpr size_t WorkerCount = 1;
};

// === Application parameterized by config ===
template<typename Cfg>
class Application {
public:
    void initialize() {
        if constexpr (Cfg::Logging::value) {
            init_logging();
        }
        if constexpr (Cfg::Metrics::value) {
            init_metrics();
        }
        if constexpr (Cfg::Tracing::value) {
            init_tracing();  // Only in debug builds
        }
        if constexpr (Cfg::DebugUI::value) {
            init_debug_ui(); // Only in debug builds
        }

        // Worker count known at compile time
        std::array<Worker, Cfg::WorkerCount> workers;
        for (auto& w : workers) w.start();
    }
};

// Select at build time:
#ifdef NDEBUG
using App = Application<ProductionConfig>;
#else
using App = Application<DebugConfig>;
#endif

// Feature checks in code:
template<typename Cfg>
void maybe_log(const char* msg) {
    if constexpr (Cfg::Logging::value) {
        printf("%s\n", msg);
    }
    // When disabled: function body is EMPTY, completely optimized away
}

```

---

## Notes

- `if constexpr` eliminates dead branches entirely — the compiler doesn't even check the unused branch's syntax for dependent code
- `static_assert` validates configuration at compile time, catching errors before runtime
- Policy classes + templates = zero-overhead customization (no vtable, fully inlined)
- Prefer `constexpr` variables over `#define` — type-safe, scoped, debugger-visible
- CMake passes configuration: `target_compile_definitions(app PRIVATE PLATFORM_EMBEDDED)`
- For embedded: compile-time configuration avoids runtime overhead and reduces binary size
- Concepts (C++20) can constrain policy types for better error messages
