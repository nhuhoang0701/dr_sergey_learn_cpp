# Use Valgrind/Helgrind for memory and thread error detection

**Category:** Tooling & Debugging  
**Item:** #237  
**Standard:** C++11  
**Reference:** <https://valgrind.org>  

---

## Topic Overview

Valgrind is a dynamic analysis framework. It runs your binary on a virtual CPU to detect errors.

| Tool | Detects |
| --- | --- |
| **Memcheck** | Memory leaks, use-after-free, uninitialized reads, buffer overflows |
| **Helgrind** | Data races, lock order violations, misuse of pthreads |
| **DRD** | Data races (alternative to Helgrind) |
| **Massif** | Heap memory profiling over time |
| **Callgrind** | Call graph profiling |

---

## Self-Assessment

### Q1: Memcheck — detect uninitialized memory use

```cpp

// uninit.cpp — compile: g++ -g -O0 -std=c++20 uninit.cpp -o uninit
#include <iostream>

int compute(int x) {
    int result;                // uninitialized!
    if (x > 10)
        result = x * 2;
    // Bug: if x <= 10, result is uninitialized
    return result;
}

int main() {
    int val = compute(5);      // uses uninitialized 'result'
    if (val > 0)               // conditional jump on uninitialized value
        std::cout << val << '\n';
}

```

```bash

$ valgrind --tool=memcheck --track-origins=yes ./uninit

# Output:
# ==12345== Conditional jump or move depends on uninitialised value(s)
# ==12345==    at 0x401234: main (uninit.cpp:12)
# ==12345==  Uninitialised value was created by a stack allocation
# ==12345==    at 0x401210: compute(int) (uninit.cpp:4)
#
# ==12345== Use of uninitialised value of size 4
# ==12345==    at 0x401234: compute(int) (uninit.cpp:8)

# Interpreting:
# - "Conditional jump" = branching on uninitialized data
# - "created by stack allocation" = local variable on stack
# - --track-origins=yes shows WHERE the uninit value came from

# Common Memcheck options:
$ valgrind --leak-check=full --show-leak-kinds=all ./myapp
$ valgrind --track-origins=yes --expensive-definedness-checks=yes ./myapp

```

### Q2: Helgrind — detect lock ordering violation

```cpp

// lock_order.cpp — compile: g++ -g -O0 -std=c++20 -pthread lock_order.cpp -o lock_order
#include <mutex>
#include <thread>
#include <iostream>

std::mutex mutex_a;
std::mutex mutex_b;
int shared_data = 0;

void thread1() {
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Lock A first
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Then lock B
    shared_data += 1;
}

void thread2() {
    std::lock_guard<std::mutex> lock_b(mutex_b);  // Lock B first!
    std::lock_guard<std::mutex> lock_a(mutex_a);  // Then lock A!
    // DEADLOCK RISK: opposite lock order from thread1!
    shared_data += 2;
}

int main() {
    std::thread t1(thread1);
    std::thread t2(thread2);
    t1.join();
    t2.join();
    std::cout << shared_data << '\n';
}

```

```bash

$ valgrind --tool=helgrind ./lock_order

# Output:
# ==12345== Thread #3: lock order violation
# ==12345==
# ==12345== Observed lock ordering:
# ==12345==   First observed acquisition order:
# ==12345==     Lock at 0x... (mutex_a) acquired at 0x401234 (lock_order.cpp:10)
# ==12345==     Lock at 0x... (mutex_b) acquired at 0x401245 (lock_order.cpp:11)
# ==12345==
# ==12345==   Conflicting acquisition order:
# ==12345==     Lock at 0x... (mutex_b) acquired at 0x401267 (lock_order.cpp:16)
# ==12345==     Lock at 0x... (mutex_a) acquired at 0x401278 (lock_order.cpp:17)
# ==12345==
# ==12345== This could result in a DEADLOCK

# Fix: always acquire locks in the same order, or use std::scoped_lock
void thread1_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // deadlock-free
    shared_data += 1;
}
void thread2_fixed() {
    std::scoped_lock lock(mutex_a, mutex_b);  // same order guaranteed
    shared_data += 2;
}

```

### Q3: Valgrind overhead vs sanitizers

| Aspect | Valgrind/Memcheck | ASan | TSan | MSan |
| --- | --- | --- | --- | --- |
| Slowdown | 20-50x | 2x | 5-15x | 3x |
| Memory overhead | 2-3x | 2-3x | 5-10x | 2-3x |
| Recompilation needed | No | Yes | Yes | Yes |
| Works on stripped binaries | Yes | No | No | No |
| Linux support | Excellent | Excellent | Excellent | Clang only |
| macOS support | Partial | Excellent | Excellent | No |
| Windows support | No | MSVC partial | No | No |
| Detects uninitialized reads | Yes | No | No | Yes (MSan) |

When to use each:

```cpp

Valgrind:                           Sanitizers:
✓ No recompilation available       ✓ Fast enough for CI
✓ Third-party binaries             ✓ Daily development
✓ Complete memory model check      ✓ Integration tests
✓ Thorough but slow analysis       ✓ Fuzz testing (needs speed)
✗ Too slow for large test suites   ✗ Require recompilation
✗ Not available on Windows         ✗ Can't check third-party binaries

```

```bash

# Recommended CI strategy:
# Fast (every commit): sanitizers
$ g++ -fsanitize=address,undefined -g ./tests.cpp -o tests && ./tests

# Thorough (nightly): Valgrind
$ valgrind --leak-check=full --error-exitcode=1 ./tests

# Thread checking:
$ g++ -fsanitize=thread -g -O1 ./tests.cpp -o tests_tsan && ./tests_tsan  # fast
$ valgrind --tool=helgrind ./tests                                         # thorough

```

---

## Notes

- Compile with `-g -O0` for best Valgrind accuracy (debug symbols, no optimization).
- `--error-exitcode=1` makes Valgrind return non-zero on errors (for CI).
- Valgrind doesn't support AVX-512 or some newer SSE instructions.
- Use `VALGRIND_DO_LEAK_CHECK` macro for programmatic leak checks.
- Helgrind detects lock order violations that may never deadlock in practice but could.
