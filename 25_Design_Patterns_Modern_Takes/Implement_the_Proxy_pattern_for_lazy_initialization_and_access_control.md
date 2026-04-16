# Implement the Proxy pattern for lazy initialization and access control

**Category:** Design Patterns — Modern Takes  
**Item:** #678  
**Reference:** <https://en.wikipedia.org/wiki/Proxy_pattern>  

---

## Topic Overview

The **Proxy pattern** wraps an object to control access to it. Modern C++ enables three key variants: **LazyProxy** (deferred construction), **AccessControlProxy** (permission checks), and **RemoteProxy** (transparent network calls). All share the same interface as the real object.

### Proxy Variants

| Variant | Purpose | Key Mechanism |
| --- | --- | --- |
| Lazy Proxy | Defer expensive construction until first use | `std::optional<T>` or `unique_ptr<T>` |
| Access Control Proxy | Check permissions before forwarding | Permission enum/token check |
| Remote Proxy | Serialize calls to remote service | HTTP/gRPC behind same interface |
| Virtual Proxy | Load on demand (images, large files) | Same as Lazy + caching |

---

## Self-Assessment

### Q1: Write a LazyProxy<T> that constructs T on first access and caches it

**Answer:**

```cpp

#include <iostream>
#include <optional>
#include <string>
#include <functional>
#include <chrono>
#include <thread>

// ═══════════ LazyProxy<T>: deferred construction ═══════════
template<typename T>
class LazyProxy {
    std::optional<T> instance_;
    std::function<T()> factory_;

public:
    explicit LazyProxy(std::function<T()> factory)
        : factory_(std::move(factory)) {}

    // Access: construct on first call, cache thereafter
    T& get() {
        if (!instance_) {
            std::cout << "  [LazyProxy] Constructing...\n";
            instance_ = factory_();
        }
        return *instance_;
    }

    const T& get() const {
        // const version — requires instance already created
        return *instance_;
    }

    T& operator*()  { return get(); }
    T* operator->() { return &get(); }

    bool is_loaded() const { return instance_.has_value(); }

    void reset() { instance_.reset(); }  // Force re-creation on next access
};

// ═══════════ Expensive-to-create resource ═══════════
class Database {
    std::string connection_;
public:
    explicit Database(const std::string& conn) : connection_(conn) {
        std::cout << "  [DB] Connecting to " << conn << "...\n";
        // Simulate slow connection
    }
    std::string query(const std::string& sql) const {
        return "Result from " + connection_ + ": " + sql;
    }
};

int main() {
    // Database NOT created here — just stores the factory
    LazyProxy<Database> db([]{ return Database("postgres://localhost/mydb"); });
    std::cout << "Proxy created. Loaded? " << db.is_loaded() << '\n';  // 0

    // First access triggers construction
    std::cout << db->query("SELECT 1") << '\n';
    // [LazyProxy] Constructing...
    // [DB] Connecting to postgres://localhost/mydb...
    // Result from postgres://localhost/mydb: SELECT 1

    // Subsequent accesses reuse cached instance
    std::cout << db->query("SELECT 2") << '\n';  // No construction
    std::cout << "Loaded? " << db.is_loaded() << '\n';  // 1
}

```

### Q2: Implement an AccessControlProxy that checks permissions before forwarding calls

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <stdexcept>

// ═══════════ Shared interface ═══════════
class FileSystem {
public:
    virtual ~FileSystem() = default;
    virtual std::string read(const std::string& path) = 0;
    virtual void write(const std::string& path, const std::string& content) = 0;
    virtual void remove(const std::string& path) = 0;
};

// ═══════════ Real implementation ═══════════
class RealFileSystem : public FileSystem {
public:
    std::string read(const std::string& path) override {
        return "Contents of " + path;
    }
    void write(const std::string& path, const std::string& content) override {
        std::cout << "  [FS] Writing " << content.size() << " bytes to " << path << '\n';
    }
    void remove(const std::string& path) override {
        std::cout << "  [FS] Deleting " << path << '\n';
    }
};

// ═══════════ Permission levels ═══════════
enum class Role { VIEWER, EDITOR, ADMIN };

// ═══════════ Access Control Proxy ═══════════
class SecureFileSystem : public FileSystem {
    RealFileSystem real_;
    Role role_;

    void require(Role min_role, const std::string& op) {
        if (static_cast<int>(role_) < static_cast<int>(min_role)) {
            throw std::runtime_error(
                "Permission denied: " + op + " requires " +
                std::to_string(static_cast<int>(min_role)));
        }
    }

public:
    explicit SecureFileSystem(Role role) : role_(role) {}

    std::string read(const std::string& path) override {
        require(Role::VIEWER, "read");
        std::cout << "  [ACL] Read allowed\n";
        return real_.read(path);
    }

    void write(const std::string& path, const std::string& content) override {
        require(Role::EDITOR, "write");
        std::cout << "  [ACL] Write allowed\n";
        real_.write(path, content);
    }

    void remove(const std::string& path) override {
        require(Role::ADMIN, "delete");
        std::cout << "  [ACL] Delete allowed\n";
        real_.remove(path);
    }
};

int main() {
    SecureFileSystem viewer_fs(Role::VIEWER);
    std::cout << viewer_fs.read("/data.txt") << '\n';  // OK

    try {
        viewer_fs.write("/data.txt", "hack");  // Permission denied
    } catch (const std::exception& e) {
        std::cout << e.what() << '\n';
    }

    SecureFileSystem admin_fs(Role::ADMIN);
    admin_fs.write("/data.txt", "updated");  // OK
    admin_fs.remove("/old.txt");             // OK
}

```

### Q3: Show a RemoteProxy that serializes calls to a network service transparently

**Answer:**

```cpp

#include <iostream>
#include <string>
#include <sstream>
#include <map>

// ═══════════ Shared interface ═══════════
class Calculator {
public:
    virtual ~Calculator() = default;
    virtual double add(double a, double b) = 0;
    virtual double multiply(double a, double b) = 0;
};

// ═══════════ Real (server-side) implementation ═══════════
class RealCalculator : public Calculator {
public:
    double add(double a, double b) override { return a + b; }
    double multiply(double a, double b) override { return a * b; }
};

// ═══════════ Simulated network transport ═══════════
class NetworkTransport {
    RealCalculator server_;  // In reality, this is on another machine
public:
    std::string send_request(const std::string& serialized) {
        // Simulate: parse request, dispatch, serialize response
        std::istringstream iss(serialized);
        std::string method;
        double a, b;
        iss >> method >> a >> b;

        double result;
        if (method == "add")
            result = server_.add(a, b);
        else if (method == "multiply")
            result = server_.multiply(a, b);
        else
            return "error: unknown method";

        return std::to_string(result);
    }
};

// ═══════════ Remote Proxy — same interface, transparent serialization ═══════════
class RemoteCalculator : public Calculator {
    NetworkTransport& transport_;

    double call(const std::string& method, double a, double b) {
        std::ostringstream request;
        request << method << " " << a << " " << b;
        std::cout << "  [Remote] Sending: " << request.str() << '\n';

        std::string response = transport_.send_request(request.str());
        std::cout << "  [Remote] Received: " << response << '\n';
        return std::stod(response);
    }

public:
    explicit RemoteCalculator(NetworkTransport& t) : transport_(t) {}

    double add(double a, double b) override {
        return call("add", a, b);
    }
    double multiply(double a, double b) override {
        return call("multiply", a, b);
    }
};

int main() {
    NetworkTransport network;
    RemoteCalculator proxy(network);

    // Client code sees Calculator interface — doesn't know it's remote
    Calculator& calc = proxy;
    std::cout << "3 + 4 = " << calc.add(3, 4) << '\n';
    std::cout << "5 * 6 = " << calc.multiply(5, 6) << '\n';
    // Output:
    //   [Remote] Sending: add 3 4
    //   [Remote] Received: 7.000000
    // 3 + 4 = 7
    //   [Remote] Sending: multiply 5 6
    //   [Remote] Received: 30.000000
    // 5 * 6 = 30
}

```

---

## Notes

- **LazyProxy** is essentially `std::optional<T>` + factory; consider `std::once_flag` for thread-safe lazy init
- **Access Control Proxy** decouples security from business logic — easier to audit and test separately
- **Remote Proxy** is the foundation of RPC frameworks (gRPC, Cap'n Proto) — same interface, network transparent
- Use `operator->()` overload to make proxies feel like smart pointers
