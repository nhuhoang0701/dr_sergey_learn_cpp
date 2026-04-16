# Use AddressSanitizer, MemorySanitizer, and ThreadSanitizer effectively

**Category:** Tooling & Debugging  
**Item:** #149  
**Standard:** C++11  
**Reference:** <https://clang.llvm.org/docs/AddressSanitizer.html>  

---

## Topic Overview

The three main sanitizers form a complementary set for catching runtime bugs:

| Sanitizer | Catches | Flag | Overhead |
| --- | --- | --- | --- |
| ASAN | Memory errors (overflow, use-after-free, leaks) | `-fsanitize=address` | ~2x |
| MSAN | Uninitialized reads | `-fsanitize=memory` | ~3x |
| TSAN | Data races, deadlocks | `-fsanitize=thread` | ~5-15x |

```cpp

Compatibility matrix:
          ASAN   MSAN   TSAN   UBSAN
  ASAN     —      ✗      ✗      ✓
  MSAN     ✗      —      ✗      ✓
  TSAN     ✗      ✗      —      ✓
  UBSAN    ✓      ✓      ✓      —

  ✓ = combinable    ✗ = mutually exclusive

```

---

## Self-Assessment

### Q1: Trigger a heap-use-after-free with ASAN

```cpp

#include <iostream>
#include <vector>

int main() {
    int* ptr = new int(42);
    std::cout << "Value: " << *ptr << '\n';

    delete ptr;  // Free the memory

    // BUG: use-after-free!
    std::cout << "After free: " << *ptr << '\n';  // ASAN catches this!

    return 0;
}
// Compile: clang++ -fsanitize=address -fno-omit-frame-pointer -g -o uaf uaf.cpp
// ASAN report:
//   ==PID==ERROR: AddressSanitizer: heap-use-after-free on address 0x...
//   READ of size 4 at 0x... thread T0
//       #0 0x... in main uaf.cpp:10
//   0x... is located 0 bytes inside of 4-byte region
//   freed by thread T0 here:
//       #0 0x... in operator delete(void*)
//       #1 0x... in main uaf.cpp:8
//   previously allocated by thread T0 here:
//       #0 0x... in operator new(unsigned long)
//       #1 0x... in main uaf.cpp:5

```

### Q2: Detect a data race with TSAN

```cpp

#include <iostream>
#include <thread>
#include <vector>

std::vector<int> shared_data;

void writer() {
    for (int i = 0; i < 1000; ++i)
        shared_data.push_back(i);  // DATA RACE: concurrent modification
}

void reader() {
    for (int i = 0; i < 1000; ++i) {
        if (!shared_data.empty())
            [[maybe_unused]] auto val = shared_data.back();  // DATA RACE
    }
}

int main() {
    std::thread t1(writer);
    std::thread t2(reader);
    t1.join();
    t2.join();
    std::cout << "Size: " << shared_data.size() << '\n';
}
// Compile: clang++ -fsanitize=thread -g -O1 -o race race.cpp -lpthread
// TSAN report:
//   WARNING: ThreadSanitizer: data race
//     Write of size 8 at 0x... by thread T1:
//       #0 std::vector<int>::push_back
//     Previous read of size 8 at 0x... by thread T2:
//       #0 std::vector<int>::back
//
// Fix: protect shared_data with std::mutex or use lock-free container

```

### Q3: Why sanitizers can't be combined arbitrarily

Each sanitizer uses fundamentally different **memory instrumentation**:

```cpp

ASAN: Shadow memory 1:8 ratio

  - Poisons redzones around allocations
  - Intercepts malloc/free with custom allocator
  - Uses ~2x memory

MSAN: Shadow memory 1:1 ratio (bit-level tracking)

  - Tracks initialization status of every bit
  - Requires all code to be instrumented
  - Uses ~3x memory

TSAN: Shadow memory 4:1 ratio (4 shadow words per 8 bytes)

  - Stores access history per memory location
  - Intercepts all synchronization primitives
  - Uses ~5-10x memory

Conflict: They all need to intercept malloc/free
and use the SAME shadow memory address space differently.

```

**Supported combinations:**

| Combination | Works? | Notes |
| --- | --- | --- |
| ASAN + UBSAN | Yes | Most useful combo |
| MSAN + UBSAN | Yes | Clang-only |
| TSAN + UBSAN | Yes | Different instrumentation layers |
| ASAN + TSAN | **No** | Both replace allocator |
| ASAN + MSAN | **No** | Shadow memory conflict |
| TSAN + MSAN | **No** | Shadow memory conflict |

**Recommended CI setup:**

```yaml

jobs:
  sanitize:
    strategy:
      matrix:
        config:

          - { name: "ASAN+UBSAN", flags: "-fsanitize=address,undefined" }
          - { name: "TSAN",       flags: "-fsanitize=thread" }
          - { name: "MSAN",       flags: "-fsanitize=memory" }

```

---

## Notes

- Always add `-fno-omit-frame-pointer -g` for readable stack traces.
- MSAN is Clang-only and requires all libraries to be instrumented.
- Use `halt_on_error=1` in sanitizer options for CI.
- UBSAN is lightweight (~5% overhead) and safe to combine with any sanitizer.
- Consider running ASAN+UBSAN in debug builds during development.
