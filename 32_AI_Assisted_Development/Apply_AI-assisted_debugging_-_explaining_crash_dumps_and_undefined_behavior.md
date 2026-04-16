# Apply AI-assisted debugging - explaining crash dumps and undefined behavior

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI assistants excel at analyzing **crash dumps**, **sanitizer output**, **core dumps**, and **undefined behavior reports**. They can decode symbol names, explain stack traces, identify root causes from ASan/UBSan/TSan output, and suggest fixes. This turns hours of debugging into minutes, especially for complex issues like memory corruption, data races, and subtle UB.

### AI Debugging Assistance Effectiveness

| Debugging Task | AI Effectiveness | Key Benefit |
| --- | --- | --- |
| Crash stack trace analysis | Excellent | Decodes mangled names, explains flow |
| ASan report interpretation | Excellent | Identifies root cause + fix |
| TSan report analysis | Good | Explains data race + synchronization fix |
| UBSan output | Excellent | Cites standard, explains why it's UB |
| Segfault in optimized code | Medium | May need debug build context |
| Heisenbugs (race conditions) | Medium | Suggests instrumentation approach |
| Memory corruption | Medium | Identifies likely cause from symptoms |

---

## Self-Assessment

### Q1: Use AI to analyze sanitizer reports

**Answer:**

```text

=== PROMPT: ASan heap-use-after-free ===

"Explain this AddressSanitizer report and suggest a fix:

=================================================================
==12345==ERROR: AddressSanitizer: heap-use-after-free on address
0x60200000efb0 at pc 0x00000051b3ef bp 0x7ffed5b0e830

READ of size 8 at 0x60200000efb0 thread T0
    #0 0x51b3ee in std::__1::basic_string<...>::size()
    #1 0x51b256 in Logger::log(std::string const&)
             (/app/src/logger.cpp:42)
    #2 0x518f3a in OrderProcessor::process(Order const&)
             (/app/src/order_processor.cpp:87)
    #3 0x5183a2 in main (/app/src/main.cpp:23)

0x60200000efb0 is located 0 bytes inside of 32-byte region
freed by thread T0 here:
    #0 0x4c3a8f in operator delete(void*)
    #1 0x51a42e in std::__1::basic_string<...>::~basic_string()
    #2 0x51a3f1 in Config::~Config()
             (/app/src/config.cpp:15)
    #3 0x5184b2 in main (/app/src/main.cpp:21)

previously allocated by thread T0 here:
    #0 0x4c3683 in operator new(unsigned long)
    #1 0x51b891 in std::__1::basic_string<...>::__init(...)
    #2 0x51b6a3 in Config::Config(std::string)
             (/app/src/config.cpp:8)
    #3 0x518312 in main (/app/src/main.cpp:18)

Context: Logger holds a reference to Config's log_path string.
Config is destroyed before Logger."


=== EXPECTED AI ANALYSIS ===

"Root cause: Object lifetime ordering issue.

1. main.cpp:18 - Config constructed, allocates log_path string
2. Logger created with reference to Config::log_path
3. main.cpp:21 - Config destroyed, log_path string freed
4. main.cpp:23 - OrderProcessor::process calls Logger::log
5. Logger::log accesses freed log_path string -> USE-AFTER-FREE

Fix options:

1. Ensure Logger is destroyed BEFORE Config (reverse declaration

   order, or use explicit init/shutdown)

2. Logger stores a copy of log_path, not a reference
3. Use shared_ptr<Config> so Config outlives Logger

Recommended fix (option 1 - RAII ordering):"

```

```cpp

// BAD: Config destroyed before Logger
int main() {
    Config config("app.conf");     // Constructed first
    Logger logger(config.log_path()); // Holds reference to config
    // ... config destroyed first (reverse order)
    // logger.log() accesses freed memory!
}

// FIX: Reverse order or use shared_ptr
int main() {
    auto config = std::make_shared<Config>("app.conf");
    Logger logger(config);  // Logger holds shared_ptr
    // Config outlives Logger
}

```

### Q2: Use AI to explain complex UB scenarios

**Answer:**

```cpp

=== PROMPT: Explain this UBSan report ===

"This code produces different results with -O0 vs -O3.
UBSan reports the issue below. Explain the root cause
and why optimization level affects the behavior:

runtime error: signed integer overflow: 2147483647 + 1
cannot be represented in type 'int'
    #0 in compute_checksum(int const*, unsigned long)
      (/app/src/checksum.cpp:12)"

```

```cpp

// The problematic code:
int compute_checksum(const int* data, size_t len) {
    int sum = 0;
    for (size_t i = 0; i < len; ++i) {
        sum += data[i];  // UB: signed integer overflow
    }
    return sum;
}

// AI explanation:
// Signed integer overflow is UNDEFINED BEHAVIOR in C++
// [expr.pre]/4: "If during evaluation an expression, the
// result is not mathematically defined or not in the range
// of representable values, the behavior is undefined."
//
// At -O0: Overflow wraps around (on x86, 2's complement)
// At -O3: Compiler assumes no UB, may optimize the loop
//   based on the assumption that sum never overflows.
//   This can eliminate bounds checks, reorder operations,
//   or produce completely different code.
//
// Fix: Use unsigned int (overflow is well-defined) or
// use __builtin_add_overflow for checked arithmetic.

// Fix option 1: unsigned arithmetic
uint32_t compute_checksum(const int* data, size_t len) {
    uint32_t sum = 0;
    for (size_t i = 0; i < len; ++i) {
        sum += static_cast<uint32_t>(data[i]);
    }
    return sum;
}

// Fix option 2: checked arithmetic
std::optional<int> compute_checksum_checked(
    const int* data, size_t len) {
    int sum = 0;
    for (size_t i = 0; i < len; ++i) {
        if (__builtin_add_overflow(sum, data[i], &sum))
            return std::nullopt;  // Overflow detected
    }
    return sum;
}

```

### Q3: Use AI to analyze ThreadSanitizer data race reports

**Answer:**

```text

=== PROMPT: TSan data race report ===

"Explain this ThreadSanitizer report and show the fix:

WARNING: ThreadSanitizer: data race (pid=1234)
  Write of size 8 at 0x7f... by thread T2:
    #0 RequestHandler::update_stats()
             request_handler.cpp:45
    #1 RequestHandler::handle()
             request_handler.cpp:32

  Previous read of size 8 at 0x7f... by thread T1:
    #0 RequestHandler::get_stats() const
             request_handler.cpp:51
    #1 MonitorThread::report()
             monitor.cpp:18

  Thread T2 is the request handling thread.
  Thread T1 is the monitoring/reporting thread."

```

```cpp

// The problematic code:
class RequestHandler {
    struct Stats {
        uint64_t total_requests = 0;    // Shared mutable state
        uint64_t total_bytes = 0;
        double avg_latency_ms = 0.0;
    };

    Stats stats_;  // No synchronization!

public:
    void handle(const Request& req) {
        auto result = process(req);
        update_stats();  // Thread T2 writes stats_
    }

    void update_stats() {
        stats_.total_requests++;     // WRITE (T2)
        stats_.total_bytes += size;
        stats_.avg_latency_ms = /*...*/;
    }

    Stats get_stats() const {
        return stats_;  // READ (T1) - data race!
    }
};

// AI-suggested fix options:

// Fix 1: Mutex (simple, correct)
class RequestHandler {
    Stats stats_;
    mutable std::mutex mtx_;
public:
    void update_stats() {
        std::lock_guard lock(mtx_);
        stats_.total_requests++;
        stats_.total_bytes += size;
    }
    Stats get_stats() const {
        std::lock_guard lock(mtx_);
        return stats_;
    }
};

// Fix 2: Atomics (higher performance, lock-free)
class RequestHandler {
    std::atomic<uint64_t> total_requests_{0};
    std::atomic<uint64_t> total_bytes_{0};
    std::atomic<double> avg_latency_{0.0};
public:
    void update_stats(uint64_t bytes, double latency) {
        total_requests_.fetch_add(1, std::memory_order_relaxed);
        total_bytes_.fetch_add(bytes, std::memory_order_relaxed);
        avg_latency_.store(latency, std::memory_order_relaxed);
    }
    Stats get_stats() const {
        return {
            total_requests_.load(std::memory_order_relaxed),
            total_bytes_.load(std::memory_order_relaxed),
            avg_latency_.load(std::memory_order_relaxed)
        };
    }
};

// Fix 3: Snapshot with double-buffer (no contention)
class RequestHandler {
    Stats current_;  // Only written by handler thread
    std::atomic<Stats*> published_{nullptr};
public:
    void publish_snapshot() {
        auto* snap = new Stats(current_);
        auto* old = published_.exchange(snap, std::memory_order_release);
        delete old;
    }
    Stats get_stats() const {
        auto* snap = published_.load(std::memory_order_acquire);
        return snap ? *snap : Stats{};
    }
};

```

---

## Notes

- When pasting sanitizer output, **include the full report** — AI needs both the error and the allocation/free stack traces
- For crash dumps, provide the **source code context** around the failing function
- AI is especially good at **decoding mangled C++ names** in stack traces
- Always **reproduce with a debug build** first (`-g -O0`) for accurate line numbers
- TSan reports show the **two conflicting accesses** — AI explains the race condition and synchronization needed
- For intermittent bugs, ask AI to suggest **instrumentation** (logging, assertions) to catch the issue
- Combine AI analysis with sanitizers: AI explains, sanitizer proves
