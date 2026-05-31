# Know the [[indeterminate]] attribute (C++26) for uninitialized variables

**Category:** Core Language Fundamentals  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2795>  

---

## Topic Overview

C++26 introduces `[[indeterminate]]` to explicitly mark variables as intentionally uninitialized, suppressing compiler warnings while keeping the UB-free guarantee via erroneous behavior.

### The Problem

Before C++26, you were stuck in an awkward position. You could leave a variable uninitialized and risk UB (plus a warning), or you could zero-initialize it and pay a real runtime cost you didn't need:

```cpp
int x;              // Uninitialized - reading x is UB (before C++26)
                    // Compiler may warn "uninitialized variable"
int y = 0;          // Initialized - but wasteful if always assigned before use
```

### The C++26 Solution

`[[indeterminate]]` is the way you tell the compiler "I know this isn't initialized yet - that's intentional, and I promise to fill it before reading it":

```cpp
// C++26: [[indeterminate]] signals intent - variable is uninitialized ON PURPOSE
[[indeterminate]] int buf[4096];  // No warning; zero-cost (no initialization)

void fill_buffer(int* buf, size_t n);
fill_buffer(buf, 4096);  // Buffer filled before any read

// Reading an [[indeterminate]] variable before assignment produces:
// - "erroneous behavior" (not UB; defined to produce an unspecified value)
// - Implementations can still diagnose it
```

The attribute generates no code at all - it's purely a signal to the compiler and any tooling listening.

### Interaction with Erroneous Behavior

Here's the important upgrade C++26 makes: it demotes uninitialized reads from full undefined behavior to "erroneous behavior":

```cpp
// C++26 changes uninitialized reads from UB to "erroneous behavior":
// - The program is valid (not UB)
// - The value is unspecified (any value of the type)
// - Implementations SHOULD diagnose (sanitizers can catch it)
// - The program won't format your hard drive

[[indeterminate]] int x;
int y = x;  // Erroneous behavior (not UB!), y gets some int value
             // Can be caught by sanitizers, but won't cause nasal demons
```

The key practical difference: the optimizer cannot exploit erroneous behavior the way it can exploit true UB.

---

## Self-Assessment

### Q1: Why not just always initialize variables

Performance-critical code (large buffers, hot loops) pays real cost for needless initialization. `memset(buf, 0, 4096)` before an immediate `read(fd, buf, 4096)` wastes cache bandwidth. `[[indeterminate]]` lets the programmer skip initialization without compiler warnings.

### Q2: What's the difference between UB and erroneous behavior

UB allows the compiler to assume the code is unreachable and optimize accordingly - sometimes called "time travel" because the effects can propagate backward through the program. Erroneous behavior guarantees the program executes normally - it just produces an unspecified value. The optimizer cannot exploit erroneous behavior like it can exploit UB.

### Q3: Where is [[indeterminate]] most useful

Performance-critical buffers that are immediately filled (I/O, DMA), output parameters filled by called functions, and large stack arrays in embedded systems where zero-initialization is costly.

---

## Notes

- `[[indeterminate]]` is purely a documentation and diagnostic attribute - it generates no code.
- C++26 redefines reading uninitialized variables as erroneous behavior (P2795).
- This attribute replaces the common pattern of `#pragma warning(suppress:...)` for intentional non-initialization.
- Expect sanitizer support (MSan, UBSan) to flag reads of `[[indeterminate]]` variables.
