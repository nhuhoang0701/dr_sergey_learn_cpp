# Design value types vs entity types and know when to use each

**Category:** OOP Design

---

## Topic Overview

C++ types fall into two fundamental design categories:

| Aspect | Value Types | Entity Types |
| --- | --- | --- |
| Identity | Defined by **content** (two Points(3,4) are equal) | Defined by **identity** (two Users with same name ≠ same user) |
| Copying | Deep copy (each copy is independent) | Non-copyable (unique identity) |
| Comparison | `==` compares content | `==` compares identity (pointer/ID) |
| Passing | By value or const& | By unique_ptr or reference |
| Storage | Stack or inline | Heap (polymorphic) or managed |
| Examples | `int`, `Point`, `Color`, `Money` | `DatabaseConnection`, `Thread`, `Window` |

### Rule of Thumb

```cpp

Value type:  "Is this thing interchangeable with another identical thing?"
             YES → value type (e.g., $10 bill is same as any other $10 bill)
             NO  → entity type (e.g., your bank account is unique)

```

---

## Self-Assessment

### Q1: Design a value type with proper semantics

**Answer:**

```cpp

#include <compare>
#include <cmath>
#include <iostream>
#include <string>
#include <format>

// ═══════════ Value type: Money ═══════════
// - Compared by content (amount + currency)
// - Copied freely
// - Immutable or value-semantic mutations
class Money {
    int64_t cents_;          // Store as integer to avoid floating-point errors
    std::string currency_;   // "USD", "EUR", etc.

public:
    Money(double amount, std::string currency)
        : cents_(static_cast<int64_t>(std::round(amount * 100)))
        , currency_(std::move(currency)) {}

    static Money from_cents(int64_t cents, std::string currency) {
        Money m(0, std::move(currency));
        m.cents_ = cents;
        return m;
    }

    double amount() const { return cents_ / 100.0; }
    int64_t cents() const { return cents_; }
    const std::string& currency() const { return currency_; }

    // Value semantics: operators return NEW values
    Money operator+(const Money& other) const {
        if (currency_ != other.currency_)
            throw std::invalid_argument("Currency mismatch");
        return from_cents(cents_ + other.cents_, currency_);
    }

    Money operator-(const Money& other) const {
        if (currency_ != other.currency_)
            throw std::invalid_argument("Currency mismatch");
        return from_cents(cents_ - other.cents_, currency_);
    }

    Money operator*(double factor) const {
        return from_cents(static_cast<int64_t>(std::round(cents_ * factor)), currency_);
    }

    // Content-based comparison (C++20 spaceship)
    auto operator<=>(const Money& other) const {
        if (currency_ != other.currency_)
            throw std::invalid_argument("Currency mismatch");
        return cents_ <=> other.cents_;
    }

    bool operator==(const Money& other) const {
        return currency_ == other.currency_ && cents_ == other.cents_;
    }

    friend std::ostream& operator<<(std::ostream& os, const Money& m) {
        return os << m.currency_ << " " << m.amount();
    }
};

int main() {
    Money a(10.50, "USD");
    Money b(3.25, "USD");
    Money c = a + b;                  // New value
    std::cout << c << "\n";           // USD 13.75

    Money d = a;                      // Copy is fine — value type
    assert(a == d);                   // Equal by content
    assert(a + b > a);                // Comparable
    return 0;
}

```

### Q2: Design an entity type with proper move-only semantics

**Answer:**

```cpp

#include <memory>
#include <string>
#include <iostream>
#include <cstdint>

// ═══════════ Entity type: DatabaseConnection ═══════════
// - Unique identity (each connection is different)
// - Non-copyable (can't duplicate a TCP connection)
// - Movable (transfer ownership)
class DatabaseConnection {
    uint64_t id_;          // Unique identity
    std::string host_;
    int fd_ = -1;          // OS resource (file descriptor)
    bool connected_ = false;

    static uint64_t next_id() {
        static uint64_t counter = 0;
        return ++counter;
    }

public:
    explicit DatabaseConnection(std::string host)
        : id_(next_id()), host_(std::move(host)) {}

    // Non-copyable! Each connection is a unique resource
    DatabaseConnection(const DatabaseConnection&) = delete;
    DatabaseConnection& operator=(const DatabaseConnection&) = delete;

    // Movable — transfer ownership
    DatabaseConnection(DatabaseConnection&& other) noexcept
        : id_(other.id_), host_(std::move(other.host_)),
          fd_(other.fd_), connected_(other.connected_) {
        other.fd_ = -1;
        other.connected_ = false;
    }

    DatabaseConnection& operator=(DatabaseConnection&& other) noexcept {
        if (this != &other) {
            disconnect();  // Release current resource
            id_ = other.id_;
            host_ = std::move(other.host_);
            fd_ = other.fd_;
            connected_ = other.connected_;
            other.fd_ = -1;
            other.connected_ = false;
        }
        return *this;
    }

    ~DatabaseConnection() { disconnect(); }

    bool connect() {
        // fd_ = ::connect(...);
        fd_ = 42;  // simulated
        connected_ = true;
        std::cout << "Connection #" << id_ << " to " << host_ << " opened\n";
        return true;
    }

    void disconnect() {
        if (connected_) {
            // ::close(fd_);
            std::cout << "Connection #" << id_ << " closed\n";
            fd_ = -1;
            connected_ = false;
        }
    }

    // Identity comparison — by ID, not by content
    bool operator==(const DatabaseConnection& other) const { return id_ == other.id_; }
    uint64_t id() const { return id_; }
};

// Entity types are typically managed via unique_ptr
class ConnectionPool {
    std::vector<std::unique_ptr<DatabaseConnection>> pool_;
public:
    DatabaseConnection& acquire(const std::string& host) {
        auto conn = std::make_unique<DatabaseConnection>(host);
        conn->connect();
        pool_.push_back(std::move(conn));
        return *pool_.back();
    }
};

```

### Q3: Show hybrid types and when the distinction matters for design decisions

**Answer:**

```cpp

#include <string>
#include <vector>
#include <memory>
#include <variant>

// ═══════════ Decision framework ═══════════
// Ask these questions:
// 1. "Are two instances with same data interchangeable?" → Value
// 2. "Does it own a unique external resource?" → Entity
// 3. "Should copying be meaningful?" → Value
// 4. "Does it have lifecycle events (open/close)?" → Entity

// ═══════════ Hybrid: Value-like entity ═══════════
// User: has identity (user_id) but also value-like data
struct UserId { uint64_t value; auto operator<=>(const UserId&) const = default; };

class User {
    UserId id_;              // Identity (entity aspect)
    std::string name_;       // Data (value aspect)
    std::string email_;

public:
    User(UserId id, std::string name, std::string email)
        : id_(id), name_(std::move(name)), email_(std::move(email)) {}

    // Entity comparison: by identity, NOT by name/email
    bool operator==(const User& other) const { return id_ == other.id_; }

    // But the data is copyable (for snapshots, DTOs, etc.)
    UserId id() const { return id_; }
    const std::string& name() const { return name_; }

    void rename(std::string new_name) { name_ = std::move(new_name); }
};

// ═══════════ Impact on container choice ═══════════
// Value types: store directly
std::vector<Money> prices;              // Copy-friendly, stack-allocated

// Entity types: store via pointer (unique ownership)
std::vector<std::unique_ptr<DatabaseConnection>> connections;

// Hybrid: depends on usage
std::vector<User> user_list;            // Snapshot/DTO — copy is fine
std::map<UserId, std::unique_ptr<User>> user_registry;  // Canonical store

// ═══════════ Impact on function signatures ═══════════
// Value: pass by value or const&
double total(std::span<const Money> prices);      // Read values

// Entity: pass by reference (never copy)
void execute(DatabaseConnection& conn, std::string_view sql);  // Use resource

// Transfer entity ownership:
void register_connection(std::unique_ptr<DatabaseConnection> conn);

```

---

## Notes

- **Default to value types** — they're simpler, thread-safe (when immutable), and cache-friendly
- Entity types almost always need Rule of Five (or Rule of Zero with unique_ptr members)
- Value types should use `auto operator<=>(const T&) const = default;` (C++20)
- In domain-driven design: **Value Objects** (Money, Address) vs **Entities** (User, Order) map directly
- The `=delete` on copy constructor is the clearest signal that a type is an entity
- Use `std::unique_ptr<Entity>` for ownership, raw pointer/reference for non-owning access
