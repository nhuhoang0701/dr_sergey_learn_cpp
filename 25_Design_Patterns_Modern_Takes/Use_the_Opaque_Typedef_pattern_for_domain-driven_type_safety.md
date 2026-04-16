# Use the Opaque Typedef pattern for domain-driven type safety

**Category:** Design Patterns — Modern Takes  
**Item:** #574  
**Reference:** <https://www.fluentcpp.com/2016/12/08/strong-types-for-strong-interfaces/>  

---

## Topic Overview

An **opaque typedef** (strong typedef) creates a new type that wraps a primitive but is **incompatible** with other types wrapping the same primitive. Unlike `using UserId = int64_t` (which is a transparent alias — still just `int64_t`), an opaque typedef prevents accidental mixing of `UserId` and `ProductId` at compile time with zero runtime overhead.

### Transparent vs Opaque Typedef

```cpp

Transparent (using):             Opaque (strong type):
  using UserId = int64_t;          struct UserId { int64_t value; };
  using ProductId = int64_t;       struct ProductId { int64_t value; };
  
  void f(UserId);                  void f(UserId);
  f(ProductId{42});  // COMPILES!  f(ProductId{42});  // COMPILE ERROR!
                     // Bug!                          // Type safe!

```

---

## Self-Assessment

### Q1: Create UserId and ProductId as distinct types backed by int64_t that cannot be accidentally mixed

**Answer:**

```cpp

#include <iostream>
#include <cstdint>
#include <string>

// ═══════════ Minimal strong typedef ═══════════
struct UserId {
    int64_t value;
    explicit UserId(int64_t v) : value(v) {}

    bool operator==(const UserId& other) const = default;
    auto operator<=>(const UserId& other) const = default;

    friend std::ostream& operator<<(std::ostream& os, const UserId& id) {
        return os << "UserId(" << id.value << ")";
    }
};

struct ProductId {
    int64_t value;
    explicit ProductId(int64_t v) : value(v) {}

    bool operator==(const ProductId& other) const = default;
    auto operator<=>(const ProductId& other) const = default;

    friend std::ostream& operator<<(std::ostream& os, const ProductId& id) {
        return os << "ProductId(" << id.value << ")";
    }
};

// ═══════════ Functions with distinct parameter types ═══════════
std::string lookup_user(UserId id) {
    return "User #" + std::to_string(id.value);
}

std::string lookup_product(ProductId id) {
    return "Product #" + std::to_string(id.value);
}

int main() {
    UserId user{42};
    ProductId product{42};

    std::cout << lookup_user(user) << '\n';        // OK
    std::cout << lookup_product(product) << '\n';  // OK

    // lookup_user(product);    // COMPILE ERROR: no conversion from ProductId to UserId
    // lookup_product(user);    // COMPILE ERROR: no conversion from UserId to ProductId

    // Even though both hold int64_t(42), they are different types
    // user == product;  // COMPILE ERROR: no operator== between UserId and ProductId

    // Comparison within same type works
    UserId user2{100};
    std::cout << "Same? " << (user == user2) << '\n';      // 0
    std::cout << "Less? " << (user < user2) << '\n';       // 1
}

```

### Q2: Use the NamedType pattern for zero-overhead strong typedefs with operator support

**Answer:**

```cpp

#include <iostream>
#include <cstdint>
#include <utility>
#include <functional>

// ═══════════ Generic NamedType: zero-overhead strong typedef ═══════════
template<typename T, typename Tag>
class NamedType {
    T value_;
public:
    explicit NamedType(T value) : value_(std::move(value)) {}

    T& get() { return value_; }
    const T& get() const { return value_; }

    bool operator==(const NamedType& other) const { return value_ == other.value_; }
    auto operator<=>(const NamedType& other) const = default;

    // Hash support for use in unordered containers
    struct Hash {
        size_t operator()(const NamedType& nt) const {
            return std::hash<T>{}(nt.value_);
        }
    };
};

// ═══════════ Distinct types via unique tag structs ═══════════
using UserId    = NamedType<int64_t, struct UserIdTag>;
using ProductId = NamedType<int64_t, struct ProductIdTag>;
using OrderId   = NamedType<int64_t, struct OrderIdTag>;
using Meters    = NamedType<double, struct MetersTag>;
using Seconds   = NamedType<double, struct SecondsTag>;

// Each is a distinct type even though underlying type is the same
static_assert(!std::is_same_v<UserId, ProductId>);
static_assert(!std::is_same_v<Meters, Seconds>);

// sizeof is identical to underlying type — zero overhead!
static_assert(sizeof(UserId) == sizeof(int64_t));
static_assert(sizeof(Meters) == sizeof(double));

void process_order(OrderId order, UserId user, ProductId product) {
    std::cout << "Order " << order.get()
              << ": user " << user.get()
              << " bought product " << product.get() << '\n';
}

int main() {
    UserId user{1};
    ProductId product{100};
    OrderId order{5000};

    process_order(order, user, product);           // OK
    // process_order(order, product, user);         // COMPILE ERROR: swapped args!
    // process_order(user, order, product);         // COMPILE ERROR: wrong type for each

    Meters distance{100.0};
    Seconds time{9.58};
    // Meters speed = distance / time;  // COMPILE ERROR: can't divide Meters by Seconds
    // Need explicit unit conversion:
    double speed = distance.get() / time.get();
    std::cout << "Speed: " << speed << " m/s\n";
}

```

### Q3: Show a compile error when passing a UserId to a function expecting a ProductId

**Answer:**

```cpp

#include <iostream>
#include <cstdint>

// Strong types
template<typename T, typename Tag>
struct StrongId {
    T value;
    explicit StrongId(T v) : value(v) {}
};

using UserId    = StrongId<int64_t, struct UserTag>;
using ProductId = StrongId<int64_t, struct ProductTag>;

void delete_product(ProductId id) {
    std::cout << "Deleting product " << id.value << '\n';
}

int main() {
    UserId user{42};
    ProductId product{99};

    delete_product(product);  // OK ✓

    // ═══════════ COMPILE ERROR DEMO ═══════════
    // Uncomment to see the error:
    // delete_product(user);

    /*
    Compiler output (GCC):
    error: could not convert 'user' from 'StrongId<long long, UserTag>'
                                     to 'StrongId<long long, ProductTag>'
    note: no known conversion for argument 1 from
          'StrongId<long long, UserTag>' to 'StrongId<long long, ProductTag>'

    Without strong types (using plain int64_t):
      delete_product(user);  // COMPILES! Silently deletes wrong thing.
      // This is a real-world bug category: parameter transposition
    */

    // ═══════════ Preventing implicit conversions ═══════════
    // delete_product(ProductId(user.value));  // OK: explicit conversion when needed
    // This forces the developer to be intentional about cross-type conversions
}

```

---

## Notes

- The Tag type is never instantiated — it exists only to make each `NamedType<int64_t, Tag>` a unique type
- `sizeof(NamedType<T, Tag>) == sizeof(T)` — the compiler optimizes away the wrapper entirely
- For arithmetic strong types (Meters, Kilograms, Seconds), consider implementing `operator+`, `operator*` with unit-correct return types
- Libraries: [NamedType](https://github.com/joboccara/NamedType), [Boost.StrongTypedef](https://www.boost.org/doc/libs/release/libs/type_traits/doc/html/boost_typetraits/reference/strong_typedef.html), [type_safe](https://github.com/foonathan/type_safe)
