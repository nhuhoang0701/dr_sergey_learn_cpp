# Debug optimized code and understand inlining, tail calls, and missing frames

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://gcc.gnu.org/onlinedocs/gcc/Debugging-Options.html>  

---

## Topic Overview

Most developers debug unoptimized builds (`-O0`) because they're easy to follow - every line maps cleanly to the corresponding source, every variable is where you expect it, and the call stack looks exactly like your source code. But production builds run at `-O2` or `-O3`, and bugs sometimes only appear there. When you have to debug an optimized binary, the experience is jarring: functions disappear from the call stack, variables are "optimized out," and breakpoints land on the wrong line.

The reason is that optimized code is a transformation of your source. The compiler is not just translating your C++ into machine instructions - it's rewriting the logic to be faster, eliminating redundancy, moving computations around, and reusing registers for multiple variables. The code that runs is genuinely different from what you wrote, and the debugger is trying to map between two different representations.

### What Optimizations Do to Debugging

Here's a quick reference for the most common optimizations and what they do to your debugging session:

| Optimization | Effect on Debugging |
| --- | --- |
| Inlining | Function "disappears" from call stack |
| Tail call elimination | Caller frame is overwritten |
| Dead code elimination | Variables "optimized away" |
| Loop unrolling | Breakpoints hit multiple times per "iteration" |
| Register allocation | Variables exist in registers, not memory |

The reason this table matters is that when something looks wrong in the debugger under `-O2`, you need to know which optimization is responsible. "Why is this variable `<optimized out>`?" - register allocation. "Why is this function not in my backtrace?" - inlining or tail call elimination. Knowing the cause tells you how to work around it.

### Practical Debugging Commands

GDB has some tools to help navigate optimized code. The key insight is that even under `-O2`, if you compiled with `-g`, the binary contains DWARF debug information describing where inlined functions came from. Modern GDB can show you inlined frames explicitly:

```bash
# GDB: show inlined functions in backtrace
(gdb) set print frame-info source-and-location
(gdb) bt   # Backtrace may skip inlined functions

# With -g -O2 -gno-column-info (GCC/Clang):
# Inlined functions shown with "[inlined]" marker
(gdb) info frame  # See "inlined into frame N"

# Examine a variable that was "optimized out"
(gdb) print my_var
# <optimized out>  <- value lives in a register, now reused

# Force a variable to be observable:
volatile int debug_val = my_var;  // Prevents optimization in debug builds

# LLDB equivalent:
(lldb) frame variable --no-summary-depth=0
```

The `<optimized out>` message is particularly common and worth understanding in detail. It doesn't mean the variable doesn't exist - it means the compiler decided the value only needed to live in a register, and by the time you set your breakpoint, that register has been reused for something else. The compiler's job is to run fast code, not to keep variables accessible for your convenience.

### Compiler Flags for Debuggable Optimized Code

You don't have to choose between "fast" and "debuggable." There's a middle ground: compile with optimization flags that speed up execution but avoid the specific transformations that destroy debuggability:

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

`-Og` is specifically designed for this: it enables every optimization that doesn't make debugging harder (constant folding, dead code elimination, etc.) while disabling the ones that do (aggressive inlining, tail calls, register allocation that kills variable visibility). It's the recommended flag for development builds where you need to debug but also want reasonable performance.

`-g3` (rather than just `-g`) includes macro definitions in the debug info, which means you can evaluate macros in the debugger instead of seeing them unexpanded.

---

## Self-Assessment

### Q1: How does `-fno-omit-frame-pointer` help debugging

The frame pointer (`rbp` on x86-64) is a register that always points to the base of the current stack frame. When this convention is followed, every frame in the call stack is linked in a chain through frame pointers, and any tool that can read memory can walk the stack reliably. Without it - which is what `-fomit-frame-pointer` (the default at `-O2`) does - the compiler is free to reuse `rbp` as a general-purpose register, which is faster but breaks stack walking. Tools like `perf`, `gdb bt`, crash reporters, and profilers all depend on frame pointer chains. With `-fno-omit-frame-pointer`, stack unwinding is fast and reliable even in fully optimized builds. This is why Google and Meta now enable it by default in production.

### Q2: Why do tail calls make debugging harder

A tail call is a function call that is the very last operation before the caller returns. The compiler's optimization here is to reuse the caller's stack frame for the callee, since the caller doesn't need its frame anymore. This is more efficient but it means the caller is literally gone from memory by the time the callee runs. When a crash occurs inside the callee, the backtrace shows only frames above the caller - the caller itself has vanished. For example, if `A()` calls `B()` and `B()` tail-calls `C()`, a crash in `C()` shows `A() -> C()` - the frame for `B()` no longer exists on the stack. You know `C()` was called somehow, but the intermediate path is missing.

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

`volatile` tells the compiler that reads and writes to this variable have observable side effects, so it can't optimize them away or keep the value solely in a register. The `asm volatile` trick is more precise - it declares to the compiler that the register holding `result` is "used" by the inline assembly, preventing the compiler from discarding it. Neither approach changes your program's logic; they just create anchor points that prevent specific optimizations during debugging.

---

## Notes

- `-Og` is the recommended optimization level for development/debug builds.
- `-fno-omit-frame-pointer` is now recommended even in production (Google, Meta use it).
- DWARF5 debug info handles inlined functions much better than DWARF4.
- Use `__attribute__((noinline))` on specific functions to prevent inlining during debugging.
