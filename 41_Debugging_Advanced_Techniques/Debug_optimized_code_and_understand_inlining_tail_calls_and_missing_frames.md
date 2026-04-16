# Debug optimized code and understand inlining, tail calls, and missing frames

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html>  

---

## Topic Overview

Debugging optimized code (`-O2`/`-O3`) is challenging because the compiler transforms your source code. The debugger shows a view that no longer matches what you wrote.

### What Optimizations Do to Debugging

| Optimization | Effect on Debugging |
| --- | --- |
| Inlining | Function "disappears" from call stack |
| Tail call elimination | Caller frame is overwritten |
| Dead code elimination | Variables "optimized away" |
| Loop unrolling | Breakpoints hit multiple times per "iteration" |
| Register allocation | Variables exist in registers, not memory |

### Practical Debugging Commands

```bash

# GDB: show inlined functions in backtrace
(gdb) set print frame-info source-and-location
(gdb) bt   # Backtrace may skip inlined functions

# With -g -O2 -gno-column-info (GCC/Clang):
# Inlined functions shown with "[inlined]" marker
(gdb) info frame  # See "inlined into frame N"

# Examine a variable that was "optimized out"
(gdb) print my_var
# <optimized out>  ← value lives in a register, now reused

# Force a variable to be observable:
volatile int debug_val = my_var;  // Prevents optimization in debug builds

# LLDB equivalent:
(lldb) frame variable --no-summary-depth=0

```

### Compiler Flags for Debuggable Optimized Code

```cmake

# Best of both worlds: optimized but debuggable
target_compile_options(myapp PRIVATE
    -O2
    -g3                  # Maximum debug info
    -fno-omit-frame-pointer  # Keep frame pointer for stack traces
    -gno-statement-frontiers  # GCC: simpler line mapping
)

# Alternatively, use -Og (optimize for debugging experience)
# Only performs optimizations that don't interfere with debugging
target_compile_options(myapp PRIVATE -Og -g3)

```

---

## Self-Assessment

### Q1: How does `-fno-omit-frame-pointer` help debugging

The frame pointer (`rbp` on x86-64) creates a linked list of stack frames. Without it (`-fomit-frame-pointer`, default at `-O2`), tools like `perf`, `gdb bt`, and crash reporters can't reliably walk the stack. With frame pointers preserved, stack unwinding is fast and reliable even in optimized code.

### Q2: Why do tail calls make debugging harder

In a tail call, the compiler reuses the caller's stack frame for the callee. When a crash occurs in the callee, the caller is gone from the stack trace. Example: `A() -> B() -> C()` where B tail-calls C: the backtrace shows `A() -> C()` — B has vanished.

### Q3: How to force a variable to be visible in optimized code

```cpp

// Method 1: volatile (prevents value from being registerized)
volatile auto debug_copy = expensive_computation();

// Method 2: asm volatile (compiler barrier)
auto result = compute();
asm volatile("" : : "r"(result) : "memory");  // Forces result to exist

// Method 3: Attribute (Clang)
[[gnu::used]] auto val = compute();

```

---

## Notes

- `-Og` is the recommended optimization level for development/debug builds.
- `-fno-omit-frame-pointer` is now recommended even in production (Google, Meta use it).
- DWARF5 debug info handles inlined functions much better than DWARF4.
- Use `__attribute__((noinline))` on specific functions to prevent inlining during debugging.
