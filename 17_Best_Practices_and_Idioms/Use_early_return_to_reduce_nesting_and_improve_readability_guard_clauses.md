# Use early return to reduce nesting and improve readability (guard clauses)

**Category:** Best Practices & Idioms  
**Item:** #404  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Res-nested>  

---

## Topic Overview

**Guard clauses** are early returns that handle error/edge cases first, leaving the main logic at the lowest nesting level. This eliminates the "arrow anti-pattern" where code is deeply nested.

```cpp

Arrow anti-pattern:              Guard clauses:
if (a) {                         if (!a) return error;
  if (b) {                       if (!b) return error;
    if (c) {                     if (!c) return error;
      if (d) {                   if (!d) return error;
        // actual work           // actual work (no nesting!)
      }
    }
  }
}

```

---

## Self-Assessment

### Q1: Refactor deeply nested code into flat guard clauses

```cpp

#include <iostream>
#include <optional>
#include <string>

struct User { std::string name; int age; bool active; };
struct Order { double amount; std::string product; };
struct Payment { bool valid; double balance; };

// BAD: 5 levels of nesting (arrow anti-pattern)
std::string process_bad(User* user, Order* order, Payment* payment) {
    if (user != nullptr) {
        if (user->active) {
            if (order != nullptr) {
                if (order->amount > 0) {
                    if (payment != nullptr && payment->valid) {
                        if (payment->balance >= order->amount) {
                            return "Order processed for " + user->name;
                        } else {
                            return "Insufficient balance";
                        }
                    } else {
                        return "Invalid payment";
                    }
                } else {
                    return "Invalid order amount";
                }
            } else {
                return "No order";
            }
        } else {
            return "User inactive";
        }
    } else {
        return "No user";
    }
}

// GOOD: flat guard clauses
std::string process_good(User* user, Order* order, Payment* payment) {
    if (!user)                              return "No user";
    if (!user->active)                      return "User inactive";
    if (!order)                             return "No order";
    if (order->amount <= 0)                 return "Invalid order amount";
    if (!payment || !payment->valid)        return "Invalid payment";
    if (payment->balance < order->amount)   return "Insufficient balance";

    return "Order processed for " + user->name;
}

int main() {
    User user{"Alice", 30, true};
    Order order{49.99, "Widget"};
    Payment payment{true, 100.0};

    std::cout << process_good(&user, &order, &payment) << '\n';

    User inactive{"Bob", 25, false};
    std::cout << process_good(&inactive, &order, &payment) << '\n';
    std::cout << process_good(nullptr, &order, &payment) << '\n';
}
// Expected output:
// Order processed for Alice
// User inactive
// No user

```

### Q2: Show that early return is compatible with RAII

```cpp

#include <fstream>
#include <iostream>
#include <memory>
#include <mutex>
#include <string>

std::mutex mtx;

class DatabaseConnection {
public:
    DatabaseConnection() { std::cout << "DB connected\n"; }
    ~DatabaseConnection() { std::cout << "DB disconnected\n"; }
    bool query(const std::string& q) { return q.size() > 0; }
};

std::string process_request(const std::string& request) {
    // RAII: all resources cleaned up on ANY return path
    std::lock_guard<std::mutex> lock(mtx);  // released on return
    auto db = std::make_unique<DatabaseConnection>();  // destroyed on return

    if (request.empty())
        return "Error: empty request";  // lock released, db destroyed

    if (request.size() > 1000)
        return "Error: request too long";  // lock released, db destroyed

    if (!db->query(request))
        return "Error: query failed";  // lock released, db destroyed

    return "Success";  // lock released, db destroyed
    // NO manual cleanup needed! RAII handles everything.
}

int main() {
    std::cout << process_request("SELECT * FROM users") << '\n';
    std::cout << "---\n";
    std::cout << process_request("") << '\n';
}
// Expected output:
// DB connected
// DB disconnected
// Success
// ---
// DB connected
// DB disconnected
// Error: empty request

```

### Q3: Explain the arrow anti-pattern and how guard clauses eliminate it

The **arrow anti-pattern** gets its name from the shape of the code when viewed from the side — it points to the right like an arrow:

```cpp

if (a) {
    if (b) {
        if (c) {
            if (d) {
                // logic here  <--- the "tip" of the arrow
            }
        }
    }
}

```

**Problems:**

1. Hard to tell which `}` matches which `if`
2. Error handling is far from the check that triggers it
3. Main logic is buried at the deepest level
4. Every new condition adds another level

**Guard clauses fix all of these:**

```cpp

if (!a) return;  // error: handle immediately
if (!b) return;  // error: handle immediately
if (!c) return;  // error: handle immediately
if (!d) return;  // error: handle immediately
// Main logic is at the TOP level, not buried deep

```

---

## Notes

- Guard clauses work perfectly with RAII — destructors run on every return path.
- Don't use early return with raw `new`/`delete` — use smart pointers instead.
- `std::optional` and `std::expected` (C++23) work great with guard clauses.
- Some style guides limit function length, making guard clauses even more valuable.
