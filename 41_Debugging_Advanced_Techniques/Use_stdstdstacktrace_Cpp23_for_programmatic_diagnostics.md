# Use std::stacktrace (C++23) for programmatic diagnostics

**Category:** Debugging Advanced Techniques  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/basic_stacktrace>  

---

## Topic Overview

Before C++23, capturing a stack trace in portable code was painful. On Linux you'd use `backtrace()` and `backtrace_symbols()` from `<execinfo.h>`. On Windows you'd call `CaptureStackBackTrace()` and then `SymFromAddr()` to resolve symbols. If you wanted something that worked on both platforms, you used Boost.Stacktrace and dealt with another dependency. Each API was different, each had slightly different behavior, and none of it was standard.

C++23's `std::stacktrace` changes this: you get a single, standardized way to capture the current call stack at runtime, iterate over frames, and print it - all in portable code. This is useful for error logging, custom assertion handlers, and exception types that carry their origin with them.

### Basic Usage

The simplest use case is capturing the current stack inside an error logging function and printing it alongside the error message. Here's what that looks like:

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

The `std::stacktrace::current()` call captures the call stack at the point it's called. Printing the resulting object (which `std::ostream` supports directly) gives you numbered frames with function names, source files, and line numbers - assuming you compiled with debug symbols.

### Integration with Exceptions

A particularly powerful pattern is embedding a stack trace inside your exception types. When an exception is thrown, the stack unwinds before you catch it, so by the time you're in the catch block the original call stack is gone. If you capture the trace at the throw site and store it in the exception object, you can examine it after catching:

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

This is the reason this pattern trips people up: they try to print the stack trace in the catch handler and wonder why it only shows the catch site. The trick is to capture the trace in the constructor, before any unwinding happens. The trace stored in `trace_` is a snapshot of the call stack at the moment the exception object was constructed, which is exactly where the throw happened.

### Individual Frame Access

If you need to do something more structured with the stack information - like filtering frames, sending them to a logging system, or storing them in a structured format - you can iterate over the individual frames and access each one's properties:

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

Each `entry` is a `std::stacktrace_entry`. The `description()` method gives you the mangled or demangled function name (depending on the implementation), `source_file()` gives you the source path, and `source_line()` gives you the line number. Note that all of these can return empty strings or zero if the binary wasn't compiled with debug information.

---

## Self-Assessment

### Q1: What are the performance implications of capturing stack traces

Capturing a stack trace is expensive - typically on the order of 1 to 10 microseconds depending on stack depth and whether symbol resolution happens eagerly or lazily. That's fast enough for error paths, but it would noticeably degrade performance if called on every function call or in a tight loop. The rule of thumb is: use `std::stacktrace` only on error paths, in assertions, and in diagnostic code. The `std::stacktrace::current(skip)` overload lets you skip the N innermost frames, which reduces the cost slightly and also removes boilerplate frames from your logging infrastructure that you don't want cluttering the output.

### Q2: How to compile for std::stacktrace support

The compiler and linker requirements vary slightly between toolchains. On GCC you need an extra library link:

```bash
# GCC 12+
g++ -std=c++23 -lstdc++_libbacktrace main.cpp

# Clang (with libc++)
clang++ -std=c++23 main.cpp

# MSVC
cl /std:c++latest main.cpp
# Debug info (/Zi) needed for file/line information
```

On all platforms, you need debug symbols (`-g` on GCC/Clang, `/Zi` on MSVC) for `source_file()` and `source_line()` to return meaningful values. Without them you'll get function addresses at best.

### Q3: What was needed before C++23

The pre-C++23 alternatives were all platform-specific, which meant portable code either picked a dependency or wrote platform dispatch by hand:

```cpp
// Platform-specific alternatives:
// Linux: backtrace() + backtrace_symbols() from <execinfo.h>
// Windows: CaptureStackBackTrace() + SymFromAddr()
// Boost: boost::stacktrace::stacktrace
// All had different APIs - C++23 unifies them
```

`boost::stacktrace` was the best portable option before C++23, and it informed the design of the standard version significantly. If you're maintaining older code that uses it, the migration to `std::stacktrace` is straightforward since the concepts are identical.

---

## Notes

- Link with `-lstdc++_libbacktrace` on GCC for actual symbol resolution.
- Stack traces are most useful when compiled with debug symbols (`-g`).
- Don't capture stack traces in hot paths - they're for error/diagnostic paths only.
- `std::stacktrace::current(skip)` skips the N innermost frames.
