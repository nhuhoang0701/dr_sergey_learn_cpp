# Know the rules for delegating constructors and their exception safety implications

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/delegating_constructor>  

---

## Topic Overview

### What Are Delegating Constructors

A **delegating constructor** is a constructor that calls another constructor of the same class in its member-initializer list. Introduced in C++11, they eliminate duplicate initialization logic:

```cpp

struct Widget {
    int x, y;
    std::string name;

    // Primary constructor — does the real work
    Widget(int x, int y, std::string name) : x(x), y(y), name(std::move(name)) {}

    // Delegating constructors — forward to primary
    Widget() : Widget(0, 0, "default") {}                    // delegates
    Widget(int x) : Widget(x, 0, "unnamed") {}               // delegates
    Widget(std::string name) : Widget(0, 0, std::move(name)) {} // delegates
};

```

### Rules

1. A delegating constructor's initializer list can **only** contain the delegation call — no other member initializers.
2. The delegated-to constructor **fully initializes** the object (including all members) before the delegating constructor body runs.
3. **Self-delegation** (directly or indirectly) is ill-formed (no diagnostic required — could infinite-loop).

```cpp

struct Bad {
    int x, y;
    // ERROR: can't mix delegation with member initializers
    // Bad(int a) : Bad(a, 0), x(a) {}  // ill-formed!
};

```

### Exception Safety: The Critical Difference

When a **delegating constructor's body** throws an exception, the destructor **IS called** because the delegated-to constructor already completed — the object is considered fully constructed.

When a **non-delegating constructor** throws, the destructor **is NOT called** because the object was never fully constructed.

```cpp

Non-delegating ctor throws → no destructor call (members destructed individually)
Delegating ctor's TARGET throws → no destructor call (object never constructed)
Delegating ctor's BODY throws → destructor IS called (object was constructed by target)

```

---

## Self-Assessment

### Q1: Write a class where three constructors delegate to a single primary constructor

```cpp

#include <iostream>
#include <string>
#include <vector>
#include <stdexcept>

class DatabaseConnection {
    std::string host_;
    int port_;
    std::string user_;
    std::string database_;
    bool connected_ = false;

public:
    // Primary constructor — accepts all parameters
    DatabaseConnection(std::string host, int port, std::string user, std::string db)
        : host_(std::move(host)), port_(port),
          user_(std::move(user)), database_(std::move(db)) {
        if (port_ <= 0 || port_ > 65535)
            throw std::invalid_argument("Invalid port: " + std::to_string(port_));
        std::cout << "Connected to " << host_ << ":" << port_
                  << " as " << user_ << " db=" << database_ << "\n";
        connected_ = true;
    }

    // Delegate 1: default database
    DatabaseConnection(std::string host, int port, std::string user)
        : DatabaseConnection(std::move(host), port, std::move(user), "default") {}

    // Delegate 2: default port and database
    DatabaseConnection(std::string host)
        : DatabaseConnection(std::move(host), 5432, "admin", "default") {}

    // Delegate 3: all defaults (localhost)
    DatabaseConnection()
        : DatabaseConnection("localhost") {}

    void query(const std::string& sql) {
        std::cout << "  Executing: " << sql << " on " << database_ << "\n";
    }

    ~DatabaseConnection() {
        if (connected_)
            std::cout << "Disconnecting from " << host_ << "\n";
    }
};

int main() {
    DatabaseConnection db1;                                    // Defaults: localhost:5432
    DatabaseConnection db2("prod-server");                     // Custom host
    DatabaseConnection db3("staging", 5433, "deploy");         // Custom host+port+user
    DatabaseConnection db4("db.example.com", 3306, "app", "mydb"); // All custom

    db1.query("SELECT 1");
    db4.query("SELECT * FROM users");

    return 0;
}

```

**Output:**

```text

Connected to localhost:5432 as admin db=default
Connected to prod-server:5432 as admin db=default
Connected to staging:5433 as deploy db=default
Connected to db.example.com:3306 as app db=mydb
  Executing: SELECT 1 on default
  Executing: SELECT * FROM users on mydb
Disconnecting from db.example.com
Disconnecting from staging
Disconnecting from prod-server
Disconnecting from localhost

```

### Q2: Exception in a delegating constructor calls the destructor

```cpp

#include <iostream>
#include <string>
#include <stdexcept>

struct Resource {
    std::string name;

    // Primary constructor
    Resource(std::string n) : name(std::move(n)) {
        std::cout << "  [Primary ctor] Constructed: " << name << "\n";
    }

    // Delegating constructor — body runs AFTER primary ctor completes
    Resource(std::string n, bool should_throw)
        : Resource(std::move(n))  // Primary ctor runs first (object is now "constructed")
    {
        std::cout << "  [Delegating ctor body] Running for: " << name << "\n";
        if (should_throw) {
            throw std::runtime_error("Error in delegating ctor!");
            // Since primary ctor completed, the object IS constructed.
            // Therefore ~Resource() WILL be called!
        }
    }

    ~Resource() {
        std::cout << "  [Destructor] Destroyed: " << name << "\n";
    }
};

int main() {
    // Case 1: No exception
    std::cout << "=== Case 1: Normal construction ===\n";
    {
        Resource r("Normal", false);
    }

    // Case 2: Exception in delegating constructor body
    std::cout << "\n=== Case 2: Exception in delegating ctor ===\n";
    try {
        Resource r("Failing", true);
    } catch (const std::exception& e) {
        std::cout << "Caught: " << e.what() << "\n";
    }

    return 0;
}

```

**Output:**

```text

=== Case 1: Normal construction ===
  [Primary ctor] Constructed: Normal
  [Delegating ctor body] Running for: Normal
  [Destructor] Destroyed: Normal

=== Case 2: Exception in delegating ctor ===
  [Primary ctor] Constructed: Failing
  [Delegating ctor body] Running for: Failing
  [Destructor] Destroyed: Failing      ← DESTRUCTOR CALLED!
Caught: Error in delegating ctor!

```

**Why:** Once the delegated-to constructor completes successfully, the object is considered **fully constructed**. If the delegating constructor's body then throws, the object must be properly cleaned up → destructor runs.

This is different from a non-delegating constructor:

```cpp

struct NoDelegation {
    std::string s;
    NoDelegation() : s("hello") {
        throw std::runtime_error("oops");
        // ~NoDelegation() is NOT called (object never fully constructed)
        // But s.~string() IS called (member was constructed)
    }
};

```

### Q3: Eliminating duplicated initialization logic

```cpp

#include <iostream>
#include <string>
#include <chrono>
#include <random>

class HttpClient {
    std::string base_url_;
    int timeout_ms_;
    int max_retries_;
    std::string auth_token_;
    bool verbose_;

    // Shared validation/setup — called once from the primary constructor
    void validate() {
        if (base_url_.empty()) throw std::invalid_argument("URL cannot be empty");
        if (timeout_ms_ <= 0) throw std::invalid_argument("Timeout must be positive");
        if (max_retries_ < 0) throw std::invalid_argument("Retries must be non-negative");
    }

public:
    // PRIMARY constructor — single point of initialization
    HttpClient(std::string url, int timeout_ms, int retries, std::string token, bool verbose)
        : base_url_(std::move(url)), timeout_ms_(timeout_ms),
          max_retries_(retries), auth_token_(std::move(token)), verbose_(verbose)
    {
        validate();
        if (verbose_) std::cout << "HttpClient created for " << base_url_ << "\n";
    }

    // Convenience: URL + timeout
    HttpClient(std::string url, int timeout_ms)
        : HttpClient(std::move(url), timeout_ms, 3, "", false) {}

    // Convenience: URL only (30s timeout, 3 retries)
    explicit HttpClient(std::string url)
        : HttpClient(std::move(url), 30000) {}

    // Convenience: with auth token
    HttpClient(std::string url, std::string token)
        : HttpClient(std::move(url), 30000, 3, std::move(token), false) {}

    void get(const std::string& path) {
        std::cout << "GET " << base_url_ << path
                  << " (timeout=" << timeout_ms_ << "ms, retries=" << max_retries_ << ")\n";
    }
};

int main() {
    HttpClient c1("https://api.example.com");
    HttpClient c2("https://api.example.com", 5000);
    HttpClient c3("https://api.example.com", "Bearer xyz123");
    HttpClient c4("https://api.example.com", 10000, 5, "Bearer abc", true);

    c1.get("/users");
    c2.get("/fast-endpoint");
    c3.get("/protected");
    c4.get("/verbose");

    return 0;
}

```

**Benefits over pre-C++11 approach:**

- **No `init()` method** — all initialization flows through one constructor.
- **No code duplication** — validation and setup logic exists once.
- **Member initializer lists are used** — more efficient than default-construct-then-assign.
- **Exception safety is centralized** — the primary constructor handles all error checking.

---

## Notes

- Delegating constructors must **only** delegate — no other member initializers in the list.
- Self-delegation (direct or indirect cycles) is undefined behavior.
- The body of a delegating constructor runs **after** the target constructor completes.
- If the delegating body throws, the destructor runs (object was fully constructed by target).
- Prefer one "primary" constructor with all parameters; delegate from simpler constructors.

```text
