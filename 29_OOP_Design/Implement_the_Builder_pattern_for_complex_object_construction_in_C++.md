# Implement the Builder pattern for complex object construction in C++

**Category:** OOP Design

---

## Topic Overview

The **Builder pattern** separates construction of a complex object from its representation. Essential when:

| Use Builder When | Example |
| --- | --- |
| Many constructor parameters | Network connection config |
| Some params optional, some required | HTTP request builder |
| Construction has validation rules | SQL query builder |
| Object should be immutable after construction | Config objects |

### Builder vs Aggregate Init vs Constructor

```cpp

Constructor:      Widget(a, b, c, d, e)       ← Easy to swap args
Aggregate init:   Widget{.x=1, .y=2}          ← C++20, limited validation
Builder:          Widget::Builder().x(1).y(2).build()  ← Validation + readability

```

---

## Self-Assessment

### Q1: Implement a fluent Builder for an HTTP request

**Answer:**

```cpp

#include <string>
#include <map>
#include <optional>
#include <stdexcept>
#include <iostream>

class HttpRequest {
public:
    class Builder;  // Forward declare

    // Getters
    const std::string& method() const { return method_; }
    const std::string& url() const { return url_; }
    const std::string& body() const { return body_; }
    const std::map<std::string, std::string>& headers() const { return headers_; }
    int timeout_ms() const { return timeout_ms_; }

private:
    HttpRequest() = default;  // Only Builder can construct
    friend class Builder;

    std::string method_;
    std::string url_;
    std::string body_;
    std::map<std::string, std::string> headers_;
    int timeout_ms_ = 30000;
};

class HttpRequest::Builder {
public:
    Builder& method(std::string m) { req_.method_ = std::move(m); return *this; }
    Builder& url(std::string u)    { req_.url_ = std::move(u); return *this; }
    Builder& body(std::string b)   { req_.body_ = std::move(b); return *this; }
    Builder& timeout(int ms)       { req_.timeout_ms_ = ms; return *this; }

    Builder& header(std::string key, std::string val) {
        req_.headers_[std::move(key)] = std::move(val);
        return *this;
    }

    // Validation at build time
    [[nodiscard]] HttpRequest build() {
        if (req_.url_.empty())
            throw std::invalid_argument("URL is required");
        if (req_.method_.empty())
            req_.method_ = "GET";
        if (req_.method_ == "POST" && req_.body_.empty())
            throw std::invalid_argument("POST requires a body");

        return std::move(req_);
    }

private:
    HttpRequest req_;
};

int main() {
    auto req = HttpRequest::Builder()
        .method("POST")
        .url("https://api.example.com/data")
        .header("Content-Type", "application/json")
        .header("Authorization", "Bearer token123")
        .body(R"({"key": "value"})")
        .timeout(5000)
        .build();

    std::cout << req.method() << " " << req.url() << "\n";
    return 0;
}

```

### Q2: Show a step builder that enforces construction order at compile time

**Answer:**

```cpp

#include <string>
#include <iostream>

class Connection {
public:
    std::string host;
    int port;
    std::string user;
    std::string password;
    bool use_ssl;
};

// Step Builder: each step returns a different type
// Enforces: host → port → credentials → build
class ConnFinal {
    Connection c_;
public:
    explicit ConnFinal(Connection c) : c_(std::move(c)) {}
    ConnFinal& ssl(bool v) { c_.use_ssl = v; return *this; }
    Connection build() { return std::move(c_); }
};

class ConnCreds {
    Connection c_;
public:
    explicit ConnCreds(Connection c) : c_(std::move(c)) {}
    ConnFinal credentials(std::string user, std::string pass) {
        c_.user = std::move(user);
        c_.password = std::move(pass);
        return ConnFinal(std::move(c_));
    }
};

class ConnPort {
    Connection c_;
public:
    explicit ConnPort(Connection c) : c_(std::move(c)) {}
    ConnCreds port(int p) {
        c_.port = p;
        return ConnCreds(std::move(c_));
    }
};

class ConnBuilder {
public:
    static ConnPort host(std::string h) {
        Connection c{};
        c.host = std::move(h);
        return ConnPort(std::move(c));
    }
};

int main() {
    // Must follow exact order: host → port → credentials → build
    auto conn = ConnBuilder::host("db.example.com")
        .port(5432)
        .credentials("admin", "secret")
        .ssl(true)
        .build();
    // ConnBuilder::host("x").build();  // WON'T COMPILE!
    std::cout << conn.host << ":" << conn.port << "\n";
    return 0;
}

```

### Q3: Implement a type-safe SQL query builder

**Answer:**

```cpp

#include <string>
#include <vector>
#include <sstream>
#include <iostream>

class SelectQuery {
public:
    SelectQuery& from(std::string table) {
        table_ = std::move(table);
        return *this;
    }
    SelectQuery& columns(std::initializer_list<std::string> cols) {
        columns_.assign(cols);
        return *this;
    }
    SelectQuery& where(std::string condition) {
        conditions_.push_back(std::move(condition));
        return *this;
    }
    SelectQuery& order_by(std::string col, bool desc = false) {
        order_ = std::move(col) + (desc ? " DESC" : " ASC");
        return *this;
    }
    SelectQuery& limit(int n) {
        limit_ = n;
        return *this;
    }

    [[nodiscard]] std::string build() const {
        std::ostringstream ss;
        ss << "SELECT ";
        if (columns_.empty()) ss << "*";
        else {
            for (size_t i = 0; i < columns_.size(); ++i) {
                if (i > 0) ss << ", ";
                ss << columns_[i];
            }
        }
        ss << " FROM " << table_;
        for (size_t i = 0; i < conditions_.size(); ++i)
            ss << (i == 0 ? " WHERE " : " AND ") << conditions_[i];
        if (!order_.empty()) ss << " ORDER BY " << order_;
        if (limit_ > 0) ss << " LIMIT " << limit_;
        return ss.str();
    }

private:
    std::string table_;
    std::vector<std::string> columns_;
    std::vector<std::string> conditions_;
    std::string order_;
    int limit_ = 0;
};

int main() {
    auto sql = SelectQuery()
        .from("users")
        .columns({"id", "name", "email"})
        .where("active = true")
        .where("role = 'admin'")
        .order_by("name")
        .limit(10)
        .build();

    std::cout << sql << "\n";
    // SELECT id, name, email FROM users WHERE active = true AND role = 'admin' ORDER BY name ASC LIMIT 10
    return 0;
}

```

---

## Notes

- Use Builder when you have **5+ parameters** or complex validation rules
- The fluent style (`builder.x(1).y(2).build()`) maximizes readability
- Step builders enforce parameter ordering at **compile time** — impossible to misuse
- Make the target class constructor private, with Builder as a friend
- C++20 designated initializers (`{.x=1, .y=2}`) can replace builders for simple aggregate types
- Always return `*this` by reference for chaining, mark `build()` as `[[nodiscard]]`
