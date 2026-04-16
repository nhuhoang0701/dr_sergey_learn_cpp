# Use std::debugging utilities (C++26): breakpoint and is_debugger_present

**Category:** Standard Library — New in C++23/26  
**Item:** #577  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/utility/breakpoint>  

---

## Topic Overview

C++26 introduces `<debugging>` with two utilities that replace decades of platform-specific hacks for programmatic debugger interaction:

| Function                        | Purpose                                               |
| --- | --- |
| `std::breakpoint()`             | Halt execution as if a debugger breakpoint was set    |
| `std::is_debugger_present()`    | Returns `true` if a debugger is attached at runtime   |
| `std::breakpoint_if_debugging()`| Calls `breakpoint()` only when debugger is attached   |

### Before vs After C++26

```cpp

Before (platform-specific):                After (portable):
─────────────────────────                  ────────────────────
#ifdef _MSC_VER                            #include <debugging>
  __debugbreak();                          std::breakpoint();
#elif defined(__GNUC__)
  __builtin_debugtrap();
#elif defined(__APPLE__)
  __builtin_trap();
#endif

```

---

## Self-Assessment

### Q1: Call std::breakpoint() to programmatically trigger a debugger break at a specific point

**Answer:**

```cpp

#include <debugging>  // C++26
#include <iostream>
#include <vector>

void process_data(const std::vector<int>& data) {
    for (size_t i = 0; i < data.size(); ++i) {
        if (data[i] < 0) {
            // Programmatic breakpoint: debugger stops here
            // In release builds without debugger: implementation-defined
            // (typically SIGTRAP/no-op)
            std::breakpoint();
            // When debugger breaks, inspect: i, data[i], call stack
        }
        std::cout << "Processing: " << data[i] << '\n';
    }
}

int main() {
    std::vector<int> values = {10, 20, -5, 30};
    process_data(values);

    // Common use: break on invariant violations during development
    int result = 42;
    if (result != 42) {
        std::breakpoint();  // Should never reach here
    }
}
// When run under debugger: execution stops at std::breakpoint()
// When run normally: behavior is implementation-defined (usually SIGTRAP → crash,
// or no-op if compiled with certain flags)

```

### Q2: Use std::is_debugger_present() to enable extra diagnostic output only when debugging

**Answer:**

```cpp

#include <debugging>  // C++26
#include <iostream>
#include <string>
#include <chrono>
#include <format>
#include <source_location>

// ═══════════ Debug-only logging ═══════════
void debug_log(const std::string& msg,
               std::source_location loc = std::source_location::current()) {
    if (std::is_debugger_present()) {
        auto now = std::chrono::system_clock::now();
        std::cerr << std::format("[DEBUG {}:{}] {} — {}\n",
                                  loc.file_name(), loc.line(), msg,
                                  now);
        // Extra: dump memory stats, validate invariants, etc.
    }
    // In production (no debugger): zero overhead — the check is a simple read
}

class Database {
    int connection_count_ = 0;
public:
    void connect(const std::string& host) {
        debug_log("Attempting connection to " + host);
        ++connection_count_;
        debug_log(std::format("Active connections: {}", connection_count_));

        if (std::is_debugger_present() && connection_count_ > 100) {
            // Alert developer of potential connection leak
            std::cerr << "WARNING: >100 connections — possible leak!\n";
            std::breakpoint_if_debugging();  // Stop only if debugger attached
        }
    }
};

int main() {
    Database db;
    db.connect("localhost:5432");

    if (std::is_debugger_present()) {
        std::cout << "Running under debugger — verbose mode enabled\n";
    } else {
        std::cout << "Production mode — minimal output\n";
    }
}

// Under debugger:
//   [DEBUG main.cpp:28] Attempting connection to localhost:5432 — 2024-...
//   [DEBUG main.cpp:30] Active connections: 1 — 2024-...
//   Running under debugger — verbose mode enabled

// Without debugger:
//   Production mode — minimal output

```

### Q3: Show a debug-only assertion that triggers std::breakpoint() before printing a detailed diagnostic

**Answer:**

```cpp

#include <debugging>  // C++26
#include <iostream>
#include <format>
#include <source_location>
#include <string_view>
#include <cstdlib>

// ═══════════ Debug assertion macro ═══════════
// Triggers breakpoint FIRST (so debugger stops at the assert site),
// then prints diagnostics

inline void debug_assert_impl(
    bool condition,
    std::string_view expr,
    std::string_view msg,
    std::source_location loc = std::source_location::current())
{
    if (!condition) {
        // Step 1: Break FIRST — debugger stops at the call site's context
        std::breakpoint_if_debugging();

        // Step 2: Print detailed diagnostic
        std::cerr << std::format(
            "\n╔══ ASSERTION FAILED ══════════════════════\n"
            "║ Expression: {}\n"
            "║ Message:    {}\n"
            "║ Location:   {}:{} in {}\n"
            "╚═════════════════════════════════════════\n",
            expr, msg, loc.file_name(), loc.line(), loc.function_name());

        // Step 3: In non-debug builds, abort
        if (!std::is_debugger_present()) {
            std::abort();
        }
        // If debugger is attached, developer can inspect and continue
    }
}

#define DEBUG_ASSERT(cond, msg) \
    debug_assert_impl((cond), #cond, (msg))

// ═══════════ Usage ═══════════
class BoundedBuffer {
    int* data_;
    size_t capacity_;
    size_t size_ = 0;

public:
    BoundedBuffer(size_t cap) : data_(new int[cap]), capacity_(cap) {}
    ~BoundedBuffer() { delete[] data_; }

    void push(int value) {
        DEBUG_ASSERT(size_ < capacity_,
            std::format("Buffer overflow: size={}, capacity={}", size_, capacity_));
        data_[size_++] = value;
    }

    int& operator[](size_t idx) {
        DEBUG_ASSERT(idx < size_,
            std::format("Index {} out of bounds [0, {})", idx, size_));
        return data_[idx];
    }
};

int main() {
    BoundedBuffer buf(3);
    buf.push(10);
    buf.push(20);
    buf.push(30);

    std::cout << "buf[1] = " << buf[1] << '\n';  // 20

    // This would trigger the assertion:
    // buf.push(40);  // ASSERTION FAILED: size_ < capacity_
    // Under debugger: breaks first, then prints diagnostic
    // Without debugger: prints diagnostic, then abort()
}

```

---

## Notes

- `<debugging>` is C++26; early support in GCC 14, Clang 19, MSVC 17.10
- `std::breakpoint_if_debugging()` = `if (is_debugger_present()) breakpoint();` — the most commonly useful form
- `is_debugger_present()` checks at **runtime**, not compile time — no `#ifdef` needed
- Overhead of `is_debugger_present()` is typically a single memory read (on Windows: `NtCurrentPeb()->BeingDebugged`)
- Unlike `assert()`, `std::breakpoint()` does not require `NDEBUG` to be unset — you control when it fires
