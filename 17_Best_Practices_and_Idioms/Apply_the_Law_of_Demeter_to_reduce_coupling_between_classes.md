# Apply the Law of Demeter to reduce coupling between classes

**Category:** Best Practices & Idioms  
**Item:** #504  
**Reference:** <https://en.wikipedia.org/wiki/Law_of_Demeter>  

---

## Topic Overview

The **Law of Demeter** (LoD), also known as the **principle of least knowledge**, states: *a method should only call methods on objects it directly knows about*. Don't "reach through" an object to access its internal collaborators.

### The Rule

A method `f()` of class `C` can call methods on:

1. `C` itself (`this->method()`)
2. `f`'s parameters
3. Objects created within `f`
4. `C`'s direct member objects

**NOT:** objects returned by other calls (“train wrecks”).

```cpp

Violation:        order.getCustomer().getAddress().getCity()
                  \___/  \_________/  \________/  \_____/
                  you      reach       reach       reach
                  know     through     through     through

Fixed:            order.getShippingCity()
                  \___/  \_____________/
                  you    directly ask

```

---

## Self-Assessment

### Q1: Identify a chain of calls `a.b().c().d()` and explain why it creates tight coupling

```cpp

#include <iostream>
#include <string>

// PROBLEM: deep coupling through method chains
class Address {
public:
    std::string city() const { return city_; }
    std::string city_ = "Seattle";
};

class Customer {
public:
    const Address& address() const { return addr_; }
    Address addr_;
};

class Order {
public:
    const Customer& customer() const { return cust_; }
    Customer cust_;
};

void print_shipping_label_bad(const Order& order) {
    // Train wreck: knows about Order -> Customer -> Address -> city
    std::string city = order.customer().address().city();
    //                 ^------- violation: 3 levels deep
    std::cout << "Ship to: " << city << '\n';
}
// WHY IS THIS BAD?
// - print_shipping_label_bad depends on Order, Customer, AND Address
// - If Address changes (e.g., city() -> city_name()), this code breaks
// - If Customer stores address differently, this code breaks
// - 3 classes coupled instead of 1

int main() {
    Order order;
    print_shipping_label_bad(order);
}
// Expected output:
// Ship to: Seattle

```

### Q2: Refactor to expose a higher-level method that hides the chain

```cpp

#include <iostream>
#include <string>

class Address {
public:
    std::string city() const { return city_; }
    std::string city_ = "Seattle";
};

class Customer {
public:
    // Customer wraps Address details
    std::string shipping_city() const { return addr_.city(); }
private:
    Address addr_;
};

class Order {
public:
    // Order exposes what callers actually need
    std::string shipping_city() const { return cust_.shipping_city(); }
private:
    Customer cust_;
};

void print_shipping_label_good(const Order& order) {
    // Only talks to Order directly
    std::string city = order.shipping_city();
    std::cout << "Ship to: " << city << '\n';
}
// Now:
// - print_shipping_label_good depends only on Order
// - Customer's internal structure can change freely
// - Address could be replaced entirely — print_shipping_label is unaffected

int main() {
    Order order;
    print_shipping_label_good(order);
}
// Expected output:
// Ship to: Seattle

```

### Q3: Explain when chaining (builder pattern, fluent interface) is intentional and acceptable

**Chaining is NOT a LoD violation when the calls are on the *same* object:**

```cpp

#include <iostream>
#include <string>
#include <sstream>

// ACCEPTABLE: builder pattern — every call returns *this
class QueryBuilder {
    std::string table_, where_, order_;
public:
    QueryBuilder& from(std::string t) { table_ = t; return *this; }
    QueryBuilder& where(std::string w) { where_ = w; return *this; }
    QueryBuilder& order_by(std::string o) { order_ = o; return *this; }

    std::string build() const {
        return "SELECT * FROM " + table_ +
               " WHERE " + where_ +
               " ORDER BY " + order_;
    }
};

int main() {
    // Every method returns the SAME object — no reaching through
    auto query = QueryBuilder()
        .from("users")
        .where("age > 18")    // returns *this, not a different object
        .order_by("name")
        .build();

    std::cout << query << '\n';

    // Also acceptable: std::ostream chaining
    std::ostringstream oss;
    oss << "Hello" << ' ' << "World";  // each << returns the same stream
    std::cout << oss.str() << '\n';
}
// Expected output:
// SELECT * FROM users WHERE age > 18 ORDER BY name
// Hello World

```

**When chaining is OK vs not:**

| Pattern | OK? | Reason |
| --- | --- | --- |
| `builder.a().b().c()` — returns `*this` | Yes | Same object throughout |
| `stream << a << b << c` — returns `stream&` | Yes | Same object throughout |
| `order.customer().address().city()` | **No** | Different objects at each level |
| `ranges::view \| filter \| transform` | Yes | DSL, each step builds a view |

---

## Notes

- LoD is a **guideline**, not a rigid law. Applying it everywhere can lead to bloated wrapper methods.
- Data objects (DTOs, structs) are generally exempt — their purpose IS direct data access.
- The real goal is **minimal coupling**: code should depend on as few types as possible.
- In C++ templates, LoD is less applicable — templates work with any type that satisfies concepts.
