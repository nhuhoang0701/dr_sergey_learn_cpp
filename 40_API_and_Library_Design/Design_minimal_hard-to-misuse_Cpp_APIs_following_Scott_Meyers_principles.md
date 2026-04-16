# Design minimal, hard-to-misuse C++ APIs following Scott Meyers' principles

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://www.aristeia.com/Papers/IEEE_Software_JulAug_2004_revised.pdf>  

---

## Topic Overview

Scott Meyers' principle: **"Make interfaces easy to use correctly and hard to use incorrectly."** This drives every API design decision.

### Key Principles

| Principle | Technique |
| --- | --- |
| Prevent misuse at compile time | Strong types, `[[nodiscard]]`, deleted overloads |
| Make defaults safe | Default-construct to valid state |
| Minimize API surface | Fewer functions = fewer mistakes |
| Express preconditions in types | Enums over bools, units in names |

### Strong Types to Prevent Parameter Confusion

```cpp

#include <chrono>
#include <string>
#include <iostream>

// BAD: easy to swap arguments
void connect_bad(const std::string& host, int port, int timeout_ms, bool use_ssl);
// connect_bad("localhost", 443, 1, true);  ← is 1 the port or timeout?

// GOOD: strong types prevent argument swapping
struct Host { std::string value; };
struct Port { uint16_t value; };
enum class UseTLS { No, Yes };

void connect_good(Host host, Port port,
                   std::chrono::milliseconds timeout,
                   UseTLS tls = UseTLS::No) {
    std::cout << "Connecting to " << host.value
              << ":" << port.value
              << " timeout=" << timeout.count() << "ms"
              << " TLS=" << (tls == UseTLS::Yes ? "yes" : "no") << "\n";
}

int main() {
    using namespace std::chrono_literals;
    connect_good(Host{"localhost"}, Port{443}, 5000ms, UseTLS::Yes);
    // Compile error if you swap Host and Port!
}

```

### [[nodiscard]] to Prevent Ignored Returns

```cpp

#include <system_error>
#include <string>

// BAD: caller can ignore the error
std::error_code remove_file_bad(const std::string& path);

// GOOD: compiler warns if error is ignored
[[nodiscard]] std::error_code remove_file(const std::string& path);

// Even better: nodiscard on the TYPE
struct [[nodiscard("Check the error code")]] Result {
    std::error_code ec;
    explicit operator bool() const { return !ec; }
};

Result create_directory(const std::string& path);

void example() {
    create_directory("/tmp/test"); // Warning: nodiscard
    if (auto r = create_directory("/tmp/test"); !r) {
        // Handle error
    }
}

```

### Deleted Overloads to Block Dangerous Conversions

```cpp

#include <cstddef>
#include <cstdint>

class Buffer {
public:
    explicit Buffer(size_t size);
    // Block implicit conversions that could be bugs
    Buffer(int) = delete;      // Prevent negative sizes
    Buffer(double) = delete;   // Prevent fractional sizes
    Buffer(nullptr_t) = delete; // Prevent null construction
};

```

---

## Self-Assessment

### Q1: Redesign a function with bool parameters to be self-documenting

```cpp

// BEFORE: What do the bools mean?
// render(scene, true, false, true);

enum class Antialiasing { Off, On };
enum class VSync { Off, On };
enum class Fullscreen { Windowed, Fullscreen };

void render(const Scene& scene,
            Antialiasing aa = Antialiasing::On,
            VSync vsync = VSync::On,
            Fullscreen fs = Fullscreen::Windowed);

// AFTER: self-documenting at call site
// render(scene, Antialiasing::On, VSync::Off, Fullscreen::Fullscreen);

```

### Q2: Show how to use factory functions instead of multi-argument constructors

```cpp

#include <chrono>
#include <memory>

class Timer {
    std::chrono::milliseconds duration_;
    bool repeating_;
    Timer(std::chrono::milliseconds d, bool r) : duration_(d), repeating_(r) {}
public:
    static Timer one_shot(std::chrono::milliseconds d) { return {d, false}; }
    static Timer repeating(std::chrono::milliseconds d) { return {d, true}; }
    // Named constructors are clearer than Timer(5000ms, true)
};

auto t = Timer::repeating(std::chrono::seconds{5});

```

### Q3: Design a builder for complex configuration

```cpp

class ServerConfig {
    std::string host_ = "0.0.0.0";
    uint16_t port_ = 8080;
    int max_connections_ = 1000;
    bool tls_ = false;

public:
    ServerConfig& host(std::string h) { host_ = std::move(h); return *this; }
    ServerConfig& port(uint16_t p) { port_ = p; return *this; }
    ServerConfig& max_connections(int n) { max_connections_ = n; return *this; }
    ServerConfig& enable_tls() { tls_ = true; return *this; }

    // All defaults are safe — you can build with zero configuration
};

auto cfg = ServerConfig{}.host("example.com").port(443).enable_tls();

```

---

## Notes

- **Prefer enums over bools** for function parameters — name documents intent at call site.
- **Use `[[nodiscard]]`** on any function whose return value must be checked (error codes, handles).
- **Delete dangerous overloads** to catch implicit conversion bugs at compile time.
- Named factory functions (static methods) are clearer than overloaded constructors.
- Follow the "principle of least surprise" — API should behave as users expect.
