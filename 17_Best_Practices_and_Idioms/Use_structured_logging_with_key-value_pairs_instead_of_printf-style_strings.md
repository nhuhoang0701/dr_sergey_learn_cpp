# Use structured logging with key-value pairs instead of printf-style strings

**Category:** Best Practices & Idioms  
**Item:** #503  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/format>  

---

## Topic Overview

**Structured logging** emits log entries as key-value pairs (often JSON) rather than free-form strings. This makes logs machine-parseable, searchable, and aggregatable.

```cpp

Unstructured: "User alice logged in from 192.168.1.1 at 2024-01-15"
Structured:   {"event":"login","user":"alice","ip":"192.168.1.1","time":"2024-01-15"}

```

---

## Self-Assessment

### Q1: Design a structured logging API with key-value pairs

```cpp

#include <iostream>
#include <string>
#include <string_view>
#include <sstream>
#include <chrono>

enum class LogLevel { Debug, Info, Warn, Error };

constexpr const char* level_str(LogLevel l) {
    switch (l) {
        case LogLevel::Debug: return "DEBUG";
        case LogLevel::Info:  return "INFO";
        case LogLevel::Warn:  return "WARN";
        case LogLevel::Error: return "ERROR";
    }
    return "UNKNOWN";
}

// Key-value pair
struct Field {
    std::string_view key;
    std::string value;
};

// Helper to create fields
Field field(std::string_view k, std::string_view v) {
    return {k, std::string(v)};
}
Field field(std::string_view k, int v) {
    return {k, std::to_string(v)};
}
Field field(std::string_view k, double v) {
    return {k, std::to_string(v)};
}

// Structured logger: outputs JSON
template<typename... Fields>
void log(LogLevel level, std::string_view message, Fields... fields) {
    std::ostringstream oss;
    oss << "{\"level\":\"" << level_str(level)
        << "\",\"msg\":\"" << message << "\"";
    ((oss << ",\"" << fields.key << "\":\"" << fields.value << "\""), ...);
    oss << "}";
    std::cout << oss.str() << '\n';
}

int main() {
    log(LogLevel::Info, "user_login",
        field("user", "alice"),
        field("ip", "192.168.1.1"),
        field("attempt", 1));

    log(LogLevel::Error, "db_connection_failed",
        field("host", "db.example.com"),
        field("port", 5432),
        field("retries", 3));

    log(LogLevel::Warn, "high_memory_usage",
        field("usage_pct", 95),
        field("threshold", 90));
}
// Expected output:
// {"level":"INFO","msg":"user_login","user":"alice","ip":"192.168.1.1","attempt":"1"}
// {"level":"ERROR","msg":"db_connection_failed","host":"db.example.com","port":"5432","retries":"3"}
// {"level":"WARN","msg":"high_memory_usage","usage_pct":"95","threshold":"90"}

```

### Q2: Compare structured vs unstructured logs for querying

```cpp

#include <iostream>

int main() {
    // === UNSTRUCTURED LOGGING ===
    // Output:
    // "2024-01-15 User alice failed login from 192.168.1.1 (attempt 3)"
    // "2024-01-15 User bob logged in from 10.0.0.1"
    // "2024-01-15 Connection to db.example.com:5432 timed out after 30s"
    //
    // To find: "all failed logins from 192.168.x.x"
    //   grep 'failed login' | grep '192.168\.'  -> fragile regex
    //   What if message format changes? Regex breaks!

    std::cout << "=== Unstructured (hard to query) ===\n";
    std::cout << "2024-01-15 User alice failed login from 192.168.1.1\n";
    std::cout << "2024-01-15 User bob logged in from 10.0.0.1\n\n";

    // === STRUCTURED LOGGING (JSON) ===
    // {"time":"2024-01-15","event":"login_failed","user":"alice","ip":"192.168.1.1"}
    // {"time":"2024-01-15","event":"login_ok","user":"bob","ip":"10.0.0.1"}
    //
    // To find: "all failed logins from 192.168.x.x"
    //   SELECT * FROM logs WHERE event='login_failed' AND ip LIKE '192.168.%'
    //   Works regardless of message format changes!

    std::cout << "=== Structured (easy to query) ===\n";
    std::cout << R"({"event":"login_failed","user":"alice","ip":"192.168.1.1"})" << '\n';
    std::cout << R"({"event":"login_ok","user":"bob","ip":"10.0.0.1"})" << '\n';
}
// Expected output:
// === Unstructured (hard to query) ===
// 2024-01-15 User alice failed login from 192.168.1.1
// 2024-01-15 User bob logged in from 10.0.0.1
//
// === Structured (easy to query) ===
// {"event":"login_failed","user":"alice","ip":"192.168.1.1"}
// {"event":"login_ok","user":"bob","ip":"10.0.0.1"}

```

### Q3: Use `std::format` as a structured log backend

```cpp

#include <iostream>
#include <format>
#include <string>
#include <string_view>

enum class Level { Info, Warn, Error };

std::string format_log(Level lvl, std::string_view event,
                       std::string_view key1, auto val1,
                       std::string_view key2, auto val2) {
    const char* level_names[] = {"INFO", "WARN", "ERROR"};
    return std::format(
        R"({{"level":"{}","event":"{}","{}":"{}","{}":"{}"}})",
        level_names[static_cast<int>(lvl)], event,
        key1, val1, key2, val2
    );
}

int main() {
    auto log1 = format_log(Level::Info, "request",
                           "path", "/api/users",
                           "status", 200);
    std::cout << log1 << '\n';

    auto log2 = format_log(Level::Error, "timeout",
                           "service", "payment",
                           "ms", 5000);
    std::cout << log2 << '\n';
}
// Expected output:
// {"level":"INFO","event":"request","path":"/api/users","status":"200"}
// {"level":"ERROR","event":"timeout","service":"payment","ms":"5000"}

```

---

## Notes

- Structured logs integrate with Elasticsearch, Splunk, Datadog, Grafana Loki.
- `std::format` (C++20) is type-safe and faster than `printf`/`sprintf`.
- Popular C++ logging libs with structured support: spdlog, quill, Boost.Log.
- Always include: timestamp, level, correlation ID, service name.
- Avoid logging sensitive data (passwords, tokens, PII).
