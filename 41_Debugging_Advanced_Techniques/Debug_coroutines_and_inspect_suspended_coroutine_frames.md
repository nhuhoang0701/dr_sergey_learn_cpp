# Debug coroutines and inspect suspended coroutine frames

**Category:** Debugging Advanced Techniques  
**Standard:** C++20  
**Reference:** <https://devblogs.microsoft.com/cppblog/cpp20-coroutine-improvements-in-visual-studio/>  

---

## Topic Overview

Coroutines are challenging to debug because the stack frame is heap-allocated, suspension/resumption splits execution across multiple resume points, and the debugger sees the coroutine "machinery" not your logic.

### Understanding Coroutine Frame Layout

```cpp

#include <coroutine>
#include <iostream>

// The compiler transforms your coroutine into:
// 1. A heap-allocated "coroutine frame" containing:
//    - The promise object
//    - All local variables that survive across suspension points
//    - A resume function pointer
//    - A destroy function pointer
//    - The current suspension point index

struct Task {
    struct promise_type {
        int result_ = 0;
        Task get_return_object() { return {std::coroutine_handle<promise_type>::from_promise(*this)}; }
        std::suspend_always initial_suspend() { return {}; }
        std::suspend_always final_suspend() noexcept { return {}; }
        void return_value(int v) { result_ = v; }
        void unhandled_exception() { std::terminate(); }
    };
    std::coroutine_handle<promise_type> handle;
};

Task compute(int x) {
    int a = x * 2;        // Stored in coroutine frame (survives co_await)
    co_await std::suspend_always{};
    int b = a + 10;       // 'a' was preserved across suspension
    co_return b;
}

```

### Debugging with GDB

```bash

# GDB: inspect coroutine frame
(gdb) print handle.address()        # Get frame address
(gdb) print *(compute::_Frame*)0x55555576c2a0  # Cast to frame type

# GCC names the frame type: FunctionName::_Frame
# Clang names it: FunctionName.Frame

# Set a breakpoint inside a coroutine:
(gdb) break compute             # Breaks at initial entry
(gdb) break compute::_Resume    # Breaks on each resume

# In MSVC debugger (Visual Studio):
# The Coroutines window shows all active coroutine frames,
# their state, and promise values

```

---

## Self-Assessment

### Q1: Why is the coroutine frame on the heap

The coroutine frame outlives the scope that created it. When `compute()` is suspended, the caller's stack frame may unwind, but the coroutine's state must persist until it's resumed (possibly from a different thread). Heap allocation ensures the frame survives across suspensions.

### Q2: How to add debugging helpers to your promise type

```cpp

struct promise_type {
    const char* name_ = "unnamed";
    int suspend_point_ = 0;

    // Track suspension points
    auto yield_value(int v) {
        ++suspend_point_;
        std::cout << "[" << name_ << "] suspended at point "
                  << suspend_point_ << " with value " << v << "\n";
        return std::suspend_always{};
    }
};

```

### Q3: What coroutine debugging support exists in IDEs

- **Visual Studio 2022**: Built-in Coroutines window showing frame state, promise values
- **CLion**: Coroutine-aware stepping (step into resumed coroutine)
- **VS Code + CodeLLDB**: Manual frame inspection via expressions
- **GDB 14+**: Improved coroutine frame printing

---

## Notes

- Use `BOOST_ASSERT` or custom logging in promise types during development.
- Coroutine frame allocation can be elided by the compiler (HALO optimization) — but this makes debugging harder.
- `std::coroutine_handle::address()` returns the raw frame pointer for manual inspection.
- Visual Studio's coroutine debugging is currently the most mature.
