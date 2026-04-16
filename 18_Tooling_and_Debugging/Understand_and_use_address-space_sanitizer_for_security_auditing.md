# Understand and use address-space sanitizer for security auditing

**Category:** Tooling & Debugging  
**Item:** #802  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

AddressSanitizer (ASAN) detects memory errors at runtime by inserting checks around every memory access. It catches bugs that are common security vulnerabilities (CWE-787, CWE-416, CWE-122).

| Bug Class | CWE | ASAN Detects? |
| --- | --- | --- |
| Heap buffer overflow | CWE-122 | Yes |
| Stack buffer overflow | CWE-121 | Yes |
| Use-after-free | CWE-416 | Yes |
| Use-after-return | CWE-562 | Yes (with flag) |
| Memory leaks | CWE-401 | Yes (LeakSanitizer) |
| Double-free | CWE-415 | Yes |

```cpp

Compile:  clang++ -fsanitize=address -fno-omit-frame-pointer -g prog.cpp
Run:      ASAN_OPTIONS=detect_leaks=1 ./a.out

```

---

## Self-Assessment

### Q1: Find a heap buffer overflow with ASAN

```cpp

#include <iostream>
#include <cstdlib>

// This function has an off-by-one bug
void fill_buffer(int* buf, int size) {
    for (int i = 0; i <= size; ++i) {  // BUG: should be i < size
        buf[i] = i * 10;
    }
}

int main() {
    const int N = 10;
    int* data = new int[N];

    fill_buffer(data, N);  // writes data[10] — heap overflow!

    // Print contents
    for (int i = 0; i < N; ++i)
        std::cout << data[i] << ' ';
    std::cout << '\n';

    delete[] data;
    return 0;
}
// Compile normally: g++ -o prog prog.cpp
//   -> may silently "work" (undefined behavior!)
//
// Compile with ASAN: clang++ -fsanitize=address -g -o prog prog.cpp
// ASAN output (approximate):
//   ==12345==ERROR: AddressSanitizer: heap-buffer-overflow
//   WRITE of size 4 at 0x... thread T0
//       #0 0x... in fill_buffer(int*, int) prog.cpp:6
//       #1 0x... in main prog.cpp:12
//   0x... is located 0 bytes to the right of 40-byte region
//   allocated by thread T0 here:
//       #0 0x... in operator new[](unsigned long)
//       #1 0x... in main prog.cpp:11

```

The **fix**: change `i <= size` to `i < size` in the loop.

### Q2: Detect memory leaks with LeakSanitizer

```cpp

#include <iostream>
#include <string>
#include <vector>

struct Connection {
    std::string host;
    int port;
    Connection(std::string h, int p) : host(std::move(h)), port(p) {
        std::cout << "Connected to " << host << ':' << port << '\n';
    }
    ~Connection() {
        std::cout << "Disconnected from " << host << ':' << port << '\n';
    }
};

void handle_request() {
    // BUG: raw new without delete — leak!
    Connection* conn = new Connection("db.example.com", 5432);
    conn->host;  // use it
    // forgot: delete conn;
}

int main() {
    for (int i = 0; i < 3; ++i)
        handle_request();  // leaks 3 connections

    std::cout << "Server shutting down\n";
    return 0;
}
// Compile: clang++ -fsanitize=address -g -o prog prog.cpp
// Run:     ASAN_OPTIONS=detect_leaks=1 ./prog
//
// Normal output:
//   Connected to db.example.com:5432
//   Connected to db.example.com:5432
//   Connected to db.example.com:5432
//   Server shutting down
//
// LSAN report at exit:
//   ==12345==ERROR: LeakSanitizer: detected memory leaks
//   Direct leak of 120 byte(s) in 3 object(s) allocated from:
//       #0 0x... in operator new(unsigned long)
//       #1 0x... in handle_request() prog.cpp:19
//   SUMMARY: AddressSanitizer: 120 byte(s) leaked in 3 allocation(s).

```

**Fix**: use `std::unique_ptr<Connection>` instead of raw `new`.

### Q3: ASAN's shadow memory model

ASAN maps every 8 bytes of application memory to 1 byte of **shadow memory**:

```cpp

Application memory:          Shadow memory:
┌────────────────────┐      ┌──────────┐
│ 8 bytes app data   │ ──→  │ 1 byte   │
│ 8 bytes app data   │ ──→  │ 1 byte   │
│ 8 bytes app data   │ ──→  │ 1 byte   │
└────────────────────┘      └──────────┘

Shadow byte values:
  0x00 = all 8 bytes accessible
  0x01..0x07 = first N bytes accessible, rest poisoned
  0xfa = heap left redzone
  0xfb = heap right redzone
  0xfd = freed heap memory
  0xf1 = stack left redzone
  0xf5 = stack partial redzone

```

**Memory overhead breakdown:**

- Shadow memory: 1/8 of virtual address space (~12.5%)
- Redzones: extra bytes around every allocation (~50-100%)
- Quarantine: freed memory kept poisoned to detect use-after-free
- **Total: ~2-3x memory overhead**

**How ASAN checks work (pseudocode):**

```cpp

// Before every memory access:
byte* shadow = (addr >> 3) + SHADOW_OFFSET;
if (*shadow != 0) {
    // Detailed check for partial accessibility
    if (*shadow <= (addr & 7) + access_size - 1)
        report_error();  // BOOM: invalid access!
}

```

**Performance impact:** ~2x slowdown (much faster than Valgrind's ~20x).

---

## Notes

- ASAN is supported by GCC (4.8+), Clang (3.1+), and MSVC (/fsanitize=address).
- Use `-fno-omit-frame-pointer` for readable stack traces.
- ASAN cannot detect uninitialized reads — use MSan for that.
- For security auditing, combine ASAN with fuzzing (libFuzzer, AFL).
- Set `ASAN_OPTIONS=detect_stack_use_after_return=1` for use-after-return detection.
