# Debug memory corruption with patterns, guard pages, and red zones

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

Memory corruption bugs (buffer overflows, use-after-free, double-free) are among the hardest to debug because symptoms appear far from the cause.

### Detection Techniques

| Technique | Overhead | Detection |
| --- | --- | --- |
| ASan (Address Sanitizer) | ~2x | Overflow, UAF, double-free, leak |
| Guard pages (mprotect) | Memory cost | Overflow/underflow at page boundary |
| Magic patterns | Minimal | Uninitialized access, free detection |
| Valgrind/memcheck | ~20x | Everything, but very slow |
| Electric Fence | Memory cost | Exact overflow detection |

### Magic Pattern Fills

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

---

## Self-Assessment

### Q1: How does Address Sanitizer detect use-after-free

ASan uses shadow memory to track the state of every 8-byte memory region. On `free()`, the region is marked as "poisoned" in shadow memory but not returned to the OS. Any subsequent access checks the shadow byte and finds it poisoned, triggering an immediate error with the allocation/free stacktraces.

### Q2: What's the difference between guard pages and red zones

Guard pages use `mprotect` to make memory pages non-accessible (PROT_NONE). Any access triggers a hardware fault (SIGSEGV). This catches overflows at exact page boundaries. Red zones are fill patterns checked on deallocation — they detect corruption but only at free time, not at the moment of corruption.

### Q3: When to use Valgrind vs ASan

Use ASan for most cases (much faster, catches most bugs). Use Valgrind when: you can't recompile (binary-only debugging), you need bit-precise uninitialized-value tracking (memcheck), or you need to detect "use of uninitialized value" in conditional branches that ASan misses.

---

## Notes

- ASan is the first tool to reach for — compile with `-fsanitize=address -fno-omit-frame-pointer`.
- MSVC debug CRT fills: `0xCD` (uninitialized), `0xDD` (freed), `0xFD` (guard).
- Combine ASan with UBSan (`-fsanitize=address,undefined`) for maximum coverage.
- Guard pages are used internally by Electric Fence and DUMA (Detect Unintended Memory Access).
