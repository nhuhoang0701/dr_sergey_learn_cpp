# Use named constants instead of magic numbers throughout the codebase

**Category:** Best Practices & Idioms  
**Item:** #407  
**Standard:** C++20  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#Res-magic>  

---

## Topic Overview

**Magic numbers** are unexplained literal values in code. They make code harder to read, maintain, and update. Replace them with `constexpr` named constants.

```cpp

BAD:  buffer.resize(1024);    // Why 1024? Max packet? Page size? Random?
GOOD: buffer.resize(kMaxPacketSize);  // Self-documenting

```

---

## Self-Assessment

### Q1: Replace magic numbers with named constants

```cpp

#include <array>
#include <iostream>
#include <vector>

// BAD: magic numbers everywhere
void process_bad(const char* data, int len) {
    if (len > 1024) return;           // what is 1024?
    if (len < 64) return;             // what is 64?
    char buffer[1024];                // again 1024
    for (int i = 0; i < 1024; ++i)    // and again...
        buffer[i] = 0;
    // If we change max packet size, we must find ALL 1024s
}

// GOOD: named constants
namespace protocol {
    constexpr int kMaxPacketSize = 1024;
    constexpr int kMinPacketSize = 64;
    constexpr int kHeaderSize = 12;
    constexpr int kMaxPayload = kMaxPacketSize - kHeaderSize;
    constexpr int kDefaultPort = 8080;
    constexpr int kMaxRetries = 3;
    constexpr double kTimeoutSeconds = 5.0;
}

void process_good(const char* data, int len) {
    if (len > protocol::kMaxPacketSize) return;
    if (len < protocol::kMinPacketSize) return;
    std::array<char, protocol::kMaxPacketSize> buffer{};
    std::cout << "Processing " << len << " bytes\n";
    std::cout << "Max payload: " << protocol::kMaxPayload << '\n';
}

int main() {
    char data[100] = {};
    process_good(data, 100);
}
// Expected output:
// Processing 100 bytes
// Max payload: 1012

```

### Q2: Show maintenance cost of magic numbers vs named constants

```cpp

#include <iostream>

// Scenario: change timeout from 5000ms to 3000ms

// BAD: magic number — must find every 5000 in the codebase
// Some 5000s might be timeouts, others might be something else!
/*
    connect(host, 5000);       // timeout?
    buffer.resize(5000);       // buffer size? NOT a timeout!
    retry_after(5000);         // timeout
    if (elapsed > 5000) ...    // timeout
*/

// GOOD: named constant — change ONE place
constexpr int kTimeoutMs = 3000;  // changed here, done!

void connect(const char* host, int timeout) {
    std::cout << "Connecting to " << host << " timeout=" << timeout << "ms\n";
}

void retry_after(int ms) {
    std::cout << "Retry after " << ms << "ms\n";
}

int main() {
    connect("server.com", kTimeoutMs);
    retry_after(kTimeoutMs);

    int elapsed = 4000;
    if (elapsed > kTimeoutMs)
        std::cout << "Timed out!\n";
}
// Expected output:
// Connecting to server.com timeout=3000ms
// Retry after 3000ms
// Timed out!

```

### Q3: Use `constexpr` with user-defined literals for self-documenting constants

```cpp

#include <chrono>
#include <iostream>

// User-defined literals for domain-specific units
constexpr long double operator""_km(long double val) { return val * 1000.0L; }
constexpr long double operator""_m(long double val) { return val; }
constexpr long double operator""_cm(long double val) { return val / 100.0L; }

constexpr long double operator""_kg(long double val) { return val; }
constexpr long double operator""_g(long double val) { return val / 1000.0L; }

int main() {
    // BAD: what unit is 42000?
    // double distance = 42000;

    // GOOD: self-documenting with UDLs
    constexpr auto marathon = 42.195_km;   // 42195.0 meters
    constexpr auto height = 180.0_cm;      // 1.8 meters
    constexpr auto weight = 75.0_kg;       // 75.0 kg
    constexpr auto coin = 5.0_g;           // 0.005 kg

    std::cout << "Marathon: " << marathon << " m\n";
    std::cout << "Height: " << height << " m\n";
    std::cout << "Weight: " << weight << " kg\n";
    std::cout << "Coin: " << coin << " kg\n";

    // std::chrono has built-in literals:
    using namespace std::chrono_literals;
    auto timeout = 500ms;
    auto delay = 2s;
    std::cout << "Timeout: " << timeout.count() << " ms\n";
    std::cout << "Delay: " << delay.count() << " s\n";
}
// Expected output:
// Marathon: 42195 m
// Height: 1.8 m
// Weight: 75 kg
// Coin: 0.005 kg
// Timeout: 500 ms
// Delay: 2 s

```

---

## Notes

- `constexpr` vs `const`: `constexpr` is evaluated at compile time, `const` may be runtime.
- Use `inline constexpr` in headers to avoid ODR violations.
- Acceptable "magic numbers": 0, 1, -1 (loop bounds), 2 (halving). Everything else should be named.
- Namespaces group related constants: `protocol::kMaxPacketSize`, `ui::kDefaultWidth`.
