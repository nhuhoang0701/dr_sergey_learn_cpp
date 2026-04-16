# Use the Service Locator pattern vs Dependency Injection in large codebases

**Category:** Project Architecture

---

## Topic Overview

**Service Locator** is a registry where objects look up their dependencies at runtime. **Dependency Injection** pushes dependencies into objects from outside. Both solve the same problem (decoupling from concrete types) but with dramatically different trade-offs. In modern C++, DI is generally preferred, but Service Locator has legitimate uses in plugin systems and legacy code.

### Pattern Comparison

| Aspect | Service Locator | Dependency Injection |
| --- | --- | --- |
| **Dependency visibility** | Hidden (ask the locator) | Explicit (in constructor) |
| **Testability** | Must configure locator | Pass mocks directly |
| **Compile-time safety** | Runtime resolution | Compile-time verification |
| **Coupling** | All code depends on locator | Only composition root wires |
| **Legacy migration** | Easy to retrofit | Requires refactoring |
| **Plugin systems** | Natural fit | Awkward with dynamic loading |

---

## Self-Assessment

### Q1: Implement a Service Locator and show its drawbacks

**Answer:**

```cpp

#include <any>
#include <typeindex>
#include <unordered_map>
#include <memory>
#include <stdexcept>

// === Service Locator ===
class ServiceLocator {
public:
    static ServiceLocator& instance() {
        static ServiceLocator sl;
        return sl;
    }

    template<typename T>
    void register_service(std::shared_ptr<T> service) {
        services_[typeid(T)] = std::move(service);
    }

    template<typename T>
    std::shared_ptr<T> resolve() {
        auto it = services_.find(typeid(T));
        if (it == services_.end())
            throw std::runtime_error("Service not registered");
        return std::any_cast<std::shared_ptr<T>>(it->second);
    }

    void clear() { services_.clear(); }

private:
    ServiceLocator() = default;
    std::unordered_map<std::type_index, std::any> services_;
};

// === Usage: dependencies are HIDDEN ===
class PaymentProcessor {
public:
    void process(double amount) {
        // BAD: dependency is invisible from the outside
        auto logger = ServiceLocator::instance().resolve<ILogger>();
        auto gateway = ServiceLocator::instance().resolve<IPaymentGateway>();
        logger->log("Processing $" + std::to_string(amount));
        gateway->charge(amount);
    }
    // Problem: the caller has no idea this needs ILogger and IPaymentGateway.
    // Reading the constructor tells you NOTHING about dependencies.
};

// === Same with DI: dependencies are EXPLICIT ===
class PaymentProcessorDI {
public:
    PaymentProcessorDI(ILogger& logger, IPaymentGateway& gateway)
        : logger_(logger), gateway_(gateway) {}

    void process(double amount) {
        logger_.log("Processing $" + std::to_string(amount));
        gateway_.charge(amount);
    }
    // CLEAR: constructor tells you exactly what this class needs.

private:
    ILogger& logger_;
    IPaymentGateway& gateway_;
};

```

### Q2: Show a legitimate use for Service Locator (plugin system)

**Answer:**

```cpp

// === Plugin system where DI is impractical ===
// Plugins are loaded at runtime via dlopen/LoadLibrary.
// We can't inject dependencies through constructors because
// the host doesn't know plugin types at compile time.

class PluginContext {
public:
    template<typename T>
    void provide(std::shared_ptr<T> service) {
        services_[typeid(T)] = std::move(service);
    }

    template<typename T>
    T& get() {
        auto it = services_.find(typeid(T));
        if (it == services_.end())
            throw std::runtime_error("Service unavailable");
        return *std::any_cast<std::shared_ptr<T>>(it->second);
    }

private:
    std::unordered_map<std::type_index, std::any> services_;
};

// Plugin interface
class IPlugin {
public:
    virtual ~IPlugin() = default;
    virtual void initialize(PluginContext& ctx) = 0;
    virtual void execute() = 0;
};

// Host provides services to plugins via locator
void load_plugins(PluginContext& ctx,
                   const std::vector<std::string>& plugin_paths) {
    ctx.provide<ILogger>(std::make_shared<FileLogger>());
    ctx.provide<IConfig>(std::make_shared<AppConfig>());

    for (const auto& path : plugin_paths) {
        auto* handle = dlopen(path.c_str(), RTLD_LAZY);
        auto create = reinterpret_cast<IPlugin*(*)()>(
            dlsym(handle, "create_plugin"));
        auto plugin = std::unique_ptr<IPlugin>(create());
        plugin->initialize(ctx);  // Plugin fetches what it needs
    }
}

// Inside a plugin:
class MyPlugin : public IPlugin {
    ILogger* logger_ = nullptr;
public:
    void initialize(PluginContext& ctx) override {
        logger_ = &ctx.get<ILogger>();  // Service Locator is natural here
    }
    void execute() override {
        logger_->log("Plugin running");
    }
};

```

### Q3: Migrate from Service Locator to DI incrementally

**Answer:**

```cpp

// === Step 1: Wrap SL calls in constructors ===
// Before:
class OrderService {
    void save(Order& o) {
        auto db = ServiceLocator::instance().resolve<IDatabase>();
        db->insert(o);
    }
};

// After step 1: Move SL call to constructor (still uses SL, but concentrated)
class OrderService {
public:
    OrderService()
        : db_(*ServiceLocator::instance().resolve<IDatabase>()) {}

    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};

// === Step 2: Add DI constructor alongside SL constructor ===
class OrderService {
public:
    // Legacy path (still works with SL)
    OrderService()
        : db_(*ServiceLocator::instance().resolve<IDatabase>()) {}

    // New path (pure DI)
    explicit OrderService(IDatabase& db) : db_(db) {}

    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};

// === Step 3: Update callers to use DI constructor ===
// Old: auto svc = OrderService();           // Uses SL
// New: auto svc = OrderService(real_db);     // Uses DI

// === Step 4: Remove default constructor once all callers migrated ===
class OrderService {
public:
    explicit OrderService(IDatabase& db) : db_(db) {}
    void save(Order& o) { db_.insert(o); }
private:
    IDatabase& db_;
};

```

---

## Notes

- **Default to Dependency Injection** for new code — explicit dependencies are easier to reason about
- Service Locator is an anti-pattern when used as a global registry for all dependencies
- Service Locator is legitimate for: plugin systems, legacy migration, dynamic service discovery
- The key smell: if you can't tell what a class needs without reading its implementation, it's a hidden dependency
- DI + composition root gives compile-time verification of the dependency graph
- In embedded C++ without dynamic loading, there's almost never a reason for Service Locator
