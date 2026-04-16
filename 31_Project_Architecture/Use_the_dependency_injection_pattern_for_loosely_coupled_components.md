# Use the dependency injection pattern for loosely coupled components

**Category:** Project Architecture

---

## Topic Overview

**Dependency Injection (DI)** passes dependencies to a class rather than having the class create them. This decouples components, enables testing with mocks, and makes the dependency graph explicit. In C++ there are three DI styles: constructor injection, setter injection, and interface injection.

### DI Styles Comparison

| Style | When to Use | Pros | Cons |
| --- | --- | --- | --- |
| **Constructor** | Required dependencies | Immutable, clear contract | Long parameter lists |
| **Setter** | Optional dependencies | Flexible, change at runtime | Nullable state |
| **Interface** | Plugin systems | Open for extension | More boilerplate |
| **Template** | Compile-time DI | Zero overhead, inlined | No runtime swap |

---

## Self-Assessment

### Q1: Implement constructor-based DI for a service

**Answer:**

```cpp

#include <memory>
#include <string>
#include <iostream>

// === Interfaces (dependencies) ===
class ILogger {
public:
    virtual ~ILogger() = default;
    virtual void log(const std::string& msg) = 0;
};

class IDatabase {
public:
    virtual ~IDatabase() = default;
    virtual void save(const std::string& key, const std::string& value) = 0;
    virtual std::string load(const std::string& key) = 0;
};

class IEmailSender {
public:
    virtual ~IEmailSender() = default;
    virtual void send(const std::string& to, const std::string& body) = 0;
};

// === Service with constructor injection ===
class UserRegistrationService {
public:
    // All dependencies injected via constructor
    UserRegistrationService(IDatabase& db, ILogger& logger, IEmailSender& email)
        : db_(db), logger_(logger), email_(email) {}

    void register_user(const std::string& name, const std::string& email_addr) {
        logger_.log("Registering user: " + name);
        db_.save("user:" + name, email_addr);
        email_.send(email_addr, "Welcome, " + name + "!");
        logger_.log("User registered successfully");
    }

private:
    IDatabase& db_;
    ILogger& logger_;
    IEmailSender& email_;
};

// === Concrete implementations ===
class ConsoleLogger : public ILogger {
public:
    void log(const std::string& msg) override {
        std::cout << "[LOG] " << msg << "\n";
    }
};

class InMemoryDb : public IDatabase {
public:
    void save(const std::string& key, const std::string& value) override {
        data_[key] = value;
    }
    std::string load(const std::string& key) override {
        return data_[key];
    }
private:
    std::unordered_map<std::string, std::string> data_;
};

class SmtpSender : public IEmailSender {
public:
    void send(const std::string& to, const std::string& body) override {
        // Real SMTP sending
    }
};

// === Composition root ===
int main() {
    ConsoleLogger logger;
    InMemoryDb db;
    SmtpSender email;

    // Wire dependencies
    UserRegistrationService service(db, logger, email);
    service.register_user("Alice", "alice@example.com");
}

```

### Q2: Use template-based DI for zero-overhead injection

**Answer:**

```cpp

#include <string>
#include <iostream>
#include <unordered_map>

// === Template DI: dependencies resolved at compile time ===
template<typename DB, typename Logger, typename Email>
class UserRegistrationServiceT {
public:
    UserRegistrationServiceT(DB& db, Logger& logger, Email& email)
        : db_(db), logger_(logger), email_(email) {}

    void register_user(const std::string& name, const std::string& addr) {
        logger_.log("Registering: " + name);
        db_.save("user:" + name, addr);
        email_.send(addr, "Welcome!");
    }

private:
    DB& db_;
    Logger& logger_;
    Email& email_;
};

// No virtual dispatch overhead — everything is inlined.
// Trade-off: can't swap implementations at runtime.

// === Usage in production ===
// auto service = UserRegistrationServiceT(real_db, real_logger, real_email);

// === Usage in tests ===
struct FakeDb {
    void save(const std::string&, const std::string&) {}
    std::string load(const std::string&) { return {}; }
};
struct FakeLogger {
    std::vector<std::string> messages;
    void log(const std::string& msg) { messages.push_back(msg); }
};
struct FakeEmail {
    int send_count = 0;
    void send(const std::string&, const std::string&) { send_count++; }
};

// Test:
// FakeDb db; FakeLogger log; FakeEmail email;
// auto svc = UserRegistrationServiceT(db, log, email);
// svc.register_user("Test", "test@test.com");
// EXPECT_EQ(email.send_count, 1);

```

### Q3: Build a simple DI container

**Answer:**

```cpp

#include <any>
#include <functional>
#include <memory>
#include <typeindex>
#include <unordered_map>
#include <stdexcept>

// === Lightweight DI Container ===
class Container {
public:
    // Register a factory for an interface type
    template<typename Interface>
    void bind(std::function<std::shared_ptr<Interface>(Container&)> factory) {
        factories_[typeid(Interface)] = [factory](Container& c) -> std::any {
            return std::static_pointer_cast<void>(factory(c));
        };
    }

    // Register a singleton
    template<typename Interface>
    void bind_singleton(std::shared_ptr<Interface> instance) {
        singletons_[typeid(Interface)] = std::static_pointer_cast<void>(instance);
    }

    // Resolve a dependency
    template<typename Interface>
    std::shared_ptr<Interface> resolve() {
        // Check singletons first
        auto sit = singletons_.find(typeid(Interface));
        if (sit != singletons_.end())
            return std::static_pointer_cast<Interface>(
                std::any_cast<std::shared_ptr<void>>(sit->second));

        // Check factories
        auto fit = factories_.find(typeid(Interface));
        if (fit == factories_.end())
            throw std::runtime_error("No binding for type");

        auto result = fit->second(*this);
        return std::static_pointer_cast<Interface>(
            std::any_cast<std::shared_ptr<void>>(result));
    }

private:
    std::unordered_map<std::type_index,
        std::function<std::any(Container&)>> factories_;
    std::unordered_map<std::type_index, std::any> singletons_;
};

// === Usage ===
void setup_production(Container& c) {
    c.bind_singleton<ILogger>(std::make_shared<ConsoleLogger>());
    c.bind<IDatabase>([](Container&) {
        return std::make_shared<InMemoryDb>();
    });
    c.bind<IEmailSender>([](Container&) {
        return std::make_shared<SmtpSender>();
    });
}

// In main:
// Container c;
// setup_production(c);
// auto logger = c.resolve<ILogger>();
// auto db = c.resolve<IDatabase>();

```

---

## Notes

- **Constructor injection** is the default choice — dependencies are explicit and immutable
- **Template DI** gives zero overhead but loses runtime flexibility — use for performance-critical paths
- A DI container is optional in C++ — manual wiring in main() is often sufficient
- If constructor parameter lists grow long (>4), it's a sign to split the class
- DI makes testing trivial: inject mocks/fakes instead of real implementations
- Avoid `shared_ptr` in DI when possible — prefer references for non-owning dependencies
- The **composition root** (main.cpp) is the single place that knows all concrete types
