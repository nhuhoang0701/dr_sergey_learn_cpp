# Debug memory corruption with patterns, guard pages, and red zones

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Memory corruption bugs - buffer overflows, use-after-free, double-free - are among the nastiest bugs you'll encounter. The reason they're so painful is that the symptom (a crash, wrong output, or silent data corruption) typically appears far from the cause. By the time your program crashes, the corrupted memory may have been read by dozens of other operations. You're left trying to work backwards from a symptom that has almost no connection to the original mistake.

The tools and techniques in this topic exist specifically to close that gap: instead of noticing corruption when it causes a crash, you detect it at the moment it happens.

### Detection Techniques

There are several approaches, each with different trade-offs between detection precision and runtime cost. Here's the lay of the land:

| Technique | Overhead | Detection |
| --- | --- | --- |
| ASan (Address Sanitizer) | ~2x | Overflow, UAF, double-free, leak |
| Guard pages (mprotect) | Memory cost | Overflow/underflow at page boundary |
| Magic patterns | Minimal | Uninitialized access, free detection |
| Valgrind/memcheck | ~20x | Everything, but very slow |
| Electric Fence | Memory cost | Exact overflow detection |

ASan is the right default choice for most development work. It's fast enough to use regularly, catches a wide range of bugs, and gives you precise stack traces. Magic patterns and guard pages are lower-level techniques you might implement yourself or see in custom allocators.

### Magic Pattern Fills

The idea behind magic patterns is simple: fill newly allocated memory with a distinctive byte value that you'd never normally see in valid data. If you later read from that memory and find the magic value, you know you're reading uninitialized or freed memory. The debug allocator below implements this approach, adding red zones around each allocation to catch overflows at deallocation time:

```cpp
#include <cstdint>
#include <cstring>

// Common debug memory patterns (used by MSVC, dlmalloc, etc.)
constexpr uint8_t UNINITIALIZED_FILL = 0xCD;  // MSVC debug CRT
constexpr uint8_t FREED_FILL         = 0xDD;  // MSVC after free
constexpr uint8_t GUARD_FILL         = 0xFD;  // Buffer boundaries
constexpr uint8_t CLEAN_FILL         = 0x00;  // Calloc

class DebugAllocator {
public:
    void* allocate(size_t size) {
        // [RED ZONE][USER DATA][RED ZONE][SIZE HEADER]
        size_t total = sizeof(size_t) + RED_ZONE + size + RED_ZONE;
        auto* raw = static_cast<uint8_t*>(::malloc(total));

        // Store size
        std::memcpy(raw, &size, sizeof(size));

        // Fill red zones with guard pattern
        std::memset(raw + sizeof(size_t), GUARD_FILL, RED_ZONE);
        std::memset(raw + sizeof(size_t) + RED_ZONE + size, GUARD_FILL, RED_ZONE);

        // Fill user memory with uninitialized pattern
        auto* user = raw + sizeof(size_t) + RED_ZONE;
        std::memset(user, UNINITIALIZED_FILL, size);
        return user;
    }

    void deallocate(void* ptr) {
        auto* user = static_cast<uint8_t*>(ptr);
        auto* raw = user - RED_ZONE - sizeof(size_t);

        size_t size;
        std::memcpy(&size, raw, sizeof(size));

        // Check red zones
        check_guard(raw + sizeof(size_t), "underflow");
        check_guard(user + size, "overflow");

        // Fill with freed pattern (detects use-after-free)
        std::memset(user, FREED_FILL, size);
        ::free(raw);
    }

private:
    static constexpr size_t RED_ZONE = 16;

    void check_guard(const uint8_t* zone, const char* name) {
        for (size_t i = 0; i < RED_ZONE; ++i) {
            if (zone[i] != GUARD_FILL) {
                std::fprintf(stderr, "MEMORY CORRUPTION: %s at offset %zu\n",
                             name, i);
                std::abort();
            }
        }
    }
};
```

Notice the fill-on-free step at the end of `deallocate`. Writing `0xDD` over the freed memory means that any subsequent read from that buffer will return `0xDDDDDDDD` instead of whatever was there before. If you then see a pointer dereference return a value built entirely of `0xDD` bytes, you know immediately that you're reading freed memory. The red zone check on `deallocate` tells you whether someone overflowed or underflowed the buffer between allocation and free.

---

## Self-Assessment

### Q1: How does Address Sanitizer detect use-after-free

ASan maintains a "shadow memory" region - a compact map where each byte of shadow memory represents the accessibility state of 8 bytes of real memory. When `free()` is called, ASan marks the corresponding shadow bytes as "poisoned" but intentionally does not return the pages to the OS. On every subsequent memory access (read or write), the compiled code checks the shadow byte for that address. If it's poisoned, ASan immediately terminates the program with a detailed error report that includes the original allocation stack trace, the free stack trace, and the current access stack trace. This is why ASan's error messages are so useful - you get all three call stacks in one report.

### Q2: What's the difference between guard pages and red zones

Guard pages use `mprotect` to mark entire memory pages as non-accessible (`PROT_NONE`). Any access to a guard page triggers a hardware memory fault (SIGSEGV on Linux), which the OS delivers synchronously at the exact instruction that crossed the boundary. This means you catch the overflow at the precise moment it occurs, not later. The downside is that page granularity is coarse (typically 4KB), so you waste memory aligning allocations to page boundaries.

Red zones are cheaper: they're just filled bytes that you check on deallocation. They can detect corruption that happened at any point between allocation and free, but only at free time - not the moment the corruption occurred. If you free the buffer a million instructions after the overflow, the corruption is still detected, but you've lost the stack trace of where it happened.

### Q3: When to use Valgrind vs ASan

Use ASan for most cases - it's roughly 20 times faster than Valgrind and catches most of the same bugs. Reach for Valgrind/memcheck when you can't recompile the program (binary-only debugging), when you need bit-precise tracking of uninitialized values flowing through conditional branches (something ASan can't do), or when you need to detect "use of uninitialized value in a conditional" rather than just a bad read.

---

## Notes

- ASan is the first tool to reach for - compile with `-fsanitize=address -fno-omit-frame-pointer`.
- MSVC debug CRT fills: `0xCD` (uninitialized), `0xDD` (freed), `0xFD` (guard).
- Combine ASan with UBSan (`-fsanitize=address,undefined`) for maximum coverage.
- Guard pages are used internally by Electric Fence and DUMA (Detect Unintended Memory Access).
