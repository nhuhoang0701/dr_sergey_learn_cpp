# Use std::stacktrace (C++23) for programmatic diagnostics

**Category:** Debugging Advanced Techniques  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/basic_stacktrace>  

---

## Topic Overview

C++23 `std::stacktrace` provides standardized stack trace capture at runtime — no more platform-specific `backtrace()` or `CaptureStackBackTrace()` calls.

### Basic Usage

```cpp

#include <stacktrace>
#include <iostream>
#include <string>

void log_error(const std::string& msg) {
    auto trace = std::stacktrace::current();
    std::cerr << "ERROR: " << msg << "\n"
              << "Stack trace:\n" << trace << "\n";
}

void inner_function() {
    log_error("something went wrong");
}

void outer_function() {
    inner_function();
}

int main() {
    outer_function();
    // Output:
    //  0# log_error at main.cpp:5
    //  1# inner_function at main.cpp:10
    //  2# outer_function at main.cpp:14
    //  3# main at main.cpp:18
}

```

### Integration with Exceptions

```cpp

#include <stacktrace>
#include <stdexcept>
#include <string>
#include <iostream>

class TracedException : public std::runtime_error {
    std::stacktrace trace_;
public:
    TracedException(const std::string& msg)
        : std::runtime_error(msg)
        , trace_(std::stacktrace::current()) {}

    const std::stacktrace& trace() const { return trace_; }
};

void risky_operation() {
    throw TracedException("file not found");
}

int main() {
    try {
        risky_operation();
    } catch (const TracedException& e) {
        std::cerr << "Exception: " << e.what() << "\n"
                  << "Thrown from:\n" << e.trace() << "\n";
    }
}

```

### Individual Frame Access

```cpp

#include <stacktrace>
#include <iostream>

void detailed_trace() {
    auto trace = std::stacktrace::current();
    for (const auto& entry : trace) {
        std::cout << "Function: " << entry.description() << "\n"
                  << "  File: " << entry.source_file() << "\n"
                  << "  Line: " << entry.source_line() << "\n";
    }
}

```

---

## Self-Assessment

### Q1: What are the performance implications of capturing stack traces

Capturing a stack trace is expensive (~1-10 microseconds depending on depth). Don't capture on every function call. Use for error paths, assertions, and diagnostics only. The `current(skip)` parameter lets you skip frames to reduce cost.

### Q2: How to compile for std::stacktrace support

```bash

# GCC 12+
g++ -std=c++23 -lstdc++_libbacktrace main.cpp

# Clang (with libc++)  
clang++ -std=c++23 main.cpp

# MSVC
cl /std:c++latest main.cpp
# Debug info (/Zi) needed for file/line information

```

### Q3: What was needed before C++23

```cpp

// Platform-specific alternatives:
// Linux: backtrace() + backtrace_symbols() from <execinfo.h>
// Windows: CaptureStackBackTrace() + SymFromAddr()
// Boost: boost::stacktrace::stacktrace
// All had different APIs — C++23 unifies them

```

---

## Notes

- Link with `-lstdc++_libbacktrace` on GCC for actual symbol resolution.
- Stack traces are most useful when compiled with debug symbols (`-g`).
- Don't capture stack traces in hot paths — they're for error/diagnostic paths only.
- `std::stacktrace::current(skip)` skips the N innermost frames.
