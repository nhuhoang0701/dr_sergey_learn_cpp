# Use strong typedefs and phantom types to prevent parameter confusion

**Category:** OOP Design

---

## Topic Overview

**Strong typedefs** create distinct types from the same underlying type, catching misuse at compile time:

```cpp

Weak (dangerous):   void transfer(int from, int to, int amount);
                    transfer(amount, to, from);  // Compiles! Silent bug!

Strong (safe):      void transfer(AccountId from, AccountId to, Money amount);
                    transfer(amount, to, from);  // COMPILE ERROR!

```

| Technique | Overhead | Conversion | Example |
| --- | --- | --- | --- |
| `enum class` | Zero | Explicit | `enum class UserId : int {};` |
| Wrapper struct | Zero (optimized away) | Explicit | `struct Meters { double value; };` |
| `std::strong_typedef` (proposed) | Zero | Explicit | Not yet in standard |
| Phantom types (template tag) | Zero | Tag-based | `Id<UserTag>` vs `Id<OrderTag>` |

---

## Self-Assessment

### Q1: Create strong typedefs using wrapper structs and enum class

**Answer:**

```cpp

#include <iostream>
#include <compare>
#include <string>
#include <functional>

// Method 1: Wrapper struct (most common)
template<typename Tag, typename T = int>
struct StrongId {
    T value;

    explicit constexpr StrongId(T v) : value(v) {}
    auto operator<=>(const StrongId&) const = default;
};

// Define distinct ID types with tags
struct UserTag {};
struct OrderTag {};
struct ProductTag {};

using UserId    = StrongId<UserTag>;
using OrderId   = StrongId<OrderTag>;
using ProductId = StrongId<ProductTag>;

// Hash support for use in containers
template<typename Tag, typename T>
struct std::hash<StrongId<Tag, T>> {
    size_t operator()(const StrongId<Tag, T>& id) const {
        return std::hash<T>{}(id.value);
    }
};

// Method 2: enum class for zero-cost distinct types
enum class Meters : int {};
enum class Feet : int {};
enum class Seconds : int {};

void navigate(Meters distance, Seconds time) {
    std::cout << "Distance: " << static_cast<int>(distance)
              << "m, Time: " << static_cast<int>(time) << "s\n";
}

void process_order(UserId user, OrderId order) {
    std::cout << "User " << user.value << " order " << order.value << "\n";
}

int main() {
    UserId uid(42);
    OrderId oid(100);
    // process_order(oid, uid);  // COMPILE ERROR! Types don't match
    process_order(uid, oid);     // OK

    // navigate(Seconds{10}, Meters{100});  // COMPILE ERROR!
    navigate(Meters{100}, Seconds{10});     // OK
    return 0;
}

```

### Q2: Build a comprehensive unit-safe type with arithmetic

**Answer:**

```cpp

#include <iostream>
#include <ratio>
#include <type_traits>

// Physical quantity with unit tag
template<typename Unit>
class Quantity {
    double value_;
public:
    constexpr explicit Quantity(double v) : value_(v) {}
    constexpr double value() const { return value_; }

    // Same-unit arithmetic
    constexpr Quantity operator+(Quantity o) const { return Quantity(value_ + o.value_); }
    constexpr Quantity operator-(Quantity o) const { return Quantity(value_ - o.value_); }
    constexpr Quantity operator*(double s) const { return Quantity(value_ * s); }
    constexpr auto operator<=>(const Quantity&) const = default;
};

// Unit tags
struct MeterTag {};
struct KilogramTag {};
struct SecondTag {};

using Meter    = Quantity<MeterTag>;
using Kilogram = Quantity<KilogramTag>;
using Second   = Quantity<SecondTag>;

// Cross-unit operations return different types
struct VelocityTag {};
using Velocity = Quantity<VelocityTag>;

constexpr Velocity operator/(Meter d, Second t) {
    return Velocity(d.value() / t.value());
}

// UDLs for ergonomics
constexpr Meter operator""_m(long double v) { return Meter(static_cast<double>(v)); }
constexpr Second operator""_s(long double v) { return Second(static_cast<double>(v)); }
constexpr Kilogram operator""_kg(long double v) { return Kilogram(static_cast<double>(v)); }

int main() {
    auto distance = 100.0_m;
    auto time = 9.58_s;
    auto speed = distance / time;
    std::cout << "Speed: " << speed.value() << " m/s\n";

    // auto bad = distance + time;  // COMPILE ERROR! Can't add meters + seconds
    auto total = distance + 50.0_m;  // OK: same units
    std::cout << "Total: " << total.value() << " m\n";
    return 0;
}

```

### Q3: Show phantom types for state-machine type safety

**Answer:**

```cpp

#include <string>
#include <iostream>

// Phantom type tags for connection states
struct Disconnected {};
struct Connected {};
struct Authenticated {};

template<typename State>
class Connection {
    std::string host_;
    std::string token_;

    // Private constructor: only transitions create new states
    Connection(std::string host, std::string token)
        : host_(std::move(host)), token_(std::move(token)) {}

    template<typename> friend class Connection;  // Allow state transitions

public:
    // Initial state: only way to create
    static Connection<Disconnected> create(std::string host) {
        return Connection<Disconnected>(std::move(host), "");
    }
};

// State transitions: each returns a different phantom type
template<>
Connection<Connected> Connection<Disconnected>::connect() {
    std::cout << "Connecting to " << host_ << "\n";
    return Connection<Connected>(host_, "");
}
// ... but we need explicit specialization. Simpler approach:

// Free functions enforce legal transitions
Connection<Connected> connect(Connection<Disconnected> c) {
    std::cout << "Connected!\n";
    return Connection<Connected>::create("host");  // Simplified
}

Connection<Authenticated> authenticate(Connection<Connected> c, const std::string& token) {
    std::cout << "Authenticated!\n";
    return Connection<Authenticated>::create("host");
}

void send_data(const Connection<Authenticated>& conn, const std::string& data) {
    std::cout << "Sending: " << data << "\n";
}

int main() {
    auto c1 = Connection<Disconnected>::create("192.168.1.1");
    // send_data(c1, "hello");  // COMPILE ERROR: not authenticated!

    auto c2 = connect(std::move(c1));
    // send_data(c2, "hello");  // COMPILE ERROR: not authenticated!

    auto c3 = authenticate(std::move(c2), "secret");
    send_data(c3, "hello");    // OK: fully authenticated
    return 0;
}

```

---

## Notes

- Strong typedefs have **zero runtime cost** — the compiler optimizes away the wrapper
- `enum class : int` is the simplest technique for opaque IDs
- Phantom types encode state-machine transitions in the type system — illegal states are compile errors
- The `<chrono>` library is the gold standard for unit-safe types in the standard library
- Don't overuse: strong types for API boundaries (public interfaces, function parameters), not internal calculations
- Proposed `std::strong_typedef` may arrive in a future standard; until then, use wrapper structs
