# Debug coroutines and inspect suspended coroutine frames

**Category:** Debugging Advanced Techniques  
**Standard:** C++20  
**Reference:** <https://devblogs.microsoft.com/cppblog/cpp20-coroutine-improvements-in-visual-studio/>  

---

## Topic Overview

Coroutines are genuinely one of the harder things to debug in modern C++. The reason is that a suspended coroutine doesn't sit on any thread's call stack - its state lives on the heap in a structure the compiler generated, and the debugger has to do extra work to find and display it. When you're used to the classic mental model of "functions live on the stack," coroutines require you to build a new mental model.

The three specific things that make coroutine debugging awkward are: the frame is heap-allocated (so normal stack inspection doesn't show it), suspension and resumption split a single logical function into multiple physical execution segments, and the debugger tends to show you the coroutine machinery rather than your own logic.

### Understanding Coroutine Frame Layout

Before you can debug coroutines effectively, you need to understand what the compiler actually produces. When you write a coroutine, the compiler rewrites it into a state machine and allocates a "coroutine frame" on the heap to hold the state. The code below shows a minimal coroutine and comments on what the compiler is actually generating:

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

The key insight here is the variable `a`. In a normal function, `a` would live on the stack and disappear when the function returns. In a coroutine, the compiler notices that `a` is used after a suspension point, so it moves `a` into the heap-allocated frame. That's why local variables in coroutines survive suspensions - and it's also why they're harder to find in a debugger.

### Debugging with GDB

To inspect a suspended coroutine in GDB, you need to know the address of its frame and then cast that address to the compiler-generated frame type. The naming convention for the frame type differs between GCC and Clang:

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

The manual cast approach is a bit clunky, but it works. You get the raw pointer from `handle.address()`, then tell GDB to interpret that memory as the compiler's frame struct. This gives you access to all the promise fields and the surviving local variables. Visual Studio's dedicated Coroutines window is considerably more convenient for this - it's one of the cases where the IDE debugger has a meaningful advantage over GDB on the command line.

---

## Self-Assessment

### Q1: Why is the coroutine frame on the heap

The coroutine frame outlives the scope that created it. When `compute()` is suspended, the caller's stack frame may unwind completely - the caller might return to its own caller, or the whole thread might move on to other work. But the coroutine's state must persist until it's resumed, which could happen from a completely different thread. Stack allocation can't survive past the return of the creating function, so heap allocation is the only option that works here.

### Q2: How to add debugging helpers to your promise type

One practical technique is to add tracking fields directly to the promise type. Since the promise is part of the coroutine frame, you can inspect these fields from the debugger whenever you have the frame address. Here's an example that tracks suspension points:

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

This gives you two things: a visible log of when and where the coroutine suspended, and fields you can inspect directly in the debugger by examining the promise object inside the frame.

### Q3: What coroutine debugging support exists in IDEs

- **Visual Studio 2022**: Built-in Coroutines window showing frame state, promise values
- **CLion**: Coroutine-aware stepping (step into resumed coroutine)
- **VS Code + CodeLLDB**: Manual frame inspection via expressions
- **GDB 14+**: Improved coroutine frame printing

---

## Notes

- Use `BOOST_ASSERT` or custom logging in promise types during development.
- Coroutine frame allocation can be elided by the compiler (HALO optimization) - but this makes debugging harder.
- `std::coroutine_handle::address()` returns the raw frame pointer for manual inspection.
- Visual Studio's coroutine debugging is currently the most mature.
