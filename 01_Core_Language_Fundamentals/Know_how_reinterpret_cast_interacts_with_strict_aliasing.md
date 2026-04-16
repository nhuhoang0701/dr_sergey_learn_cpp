# Know how reinterpret_cast interacts with strict aliasing

**Category:** Core Language Fundamentals  
**Standard:** C++11/17  
**Reference:** <https://en.cppreference.com/w/cpp/language/reinterpret_cast>  

---

## Topic Overview

### What Is Strict Aliasing

The **strict aliasing rule** states that accessing an object through a pointer/reference of a **different type** than its actual type is **undefined behavior (UB)**, with a few exceptions. This rule enables the compiler to generate faster code by assuming that pointers of unrelated types don't alias (point to the same memory).

### The Rule (Simplified)

An object of type `T` may only be accessed through:

1. A pointer/reference of type `T` (possibly cv-qualified)
2. A pointer/reference to a **signed/unsigned variant** of `T`
3. A pointer/reference to `char`, `unsigned char`, or `std::byte` (C++17)
4. A pointer/reference to a **base class** of `T` (for class types)
5. An aggregate or union containing one of the above types

Everything else is **UB**.

### Why Does reinterpret_cast Matter

`reinterpret_cast` converts between unrelated pointer types. It **compiles**, but the resulting pointer may violate strict aliasing when **dereferenced**:

```cpp

float f = 3.14f;
int* ip = reinterpret_cast<int*>(&f);
int bits = *ip;  // UB! Accessing float through int* violates strict aliasing

```

The cast itself is legal — the UB occurs on **access**.

### How the Compiler Exploits Strict Aliasing

```cpp

void optimize_example(int* ip, float* fp) {
    *ip = 42;
    *fp = 3.14f;
    // The compiler ASSUMES ip and fp don't alias (different types)
    // So it can reorder the stores or cache *ip in a register
    std::cout << *ip;  // Compiler may output 42 even if ip == (int*)fp
}

```

With strict aliasing, the compiler generates:

```asm

mov [rdi], 42        ; store to ip
movss [rsi], 3.14    ; store to fp
mov eax, 42          ; loads 42 directly (cached, skips memory read!)

```

Without the rule, the compiler must reload from memory every time.

### The Correct Way: `std::memcpy` or `std::bit_cast`

```cpp

#include <cstring>
#include <bit>       // C++20
#include <iostream>

float int_bits_to_float_safe(int i) {
    float f;
    std::memcpy(&f, &i, sizeof(f));  // SAFE: no aliasing violation
    return f;
}

// Even better in C++20:
float int_bits_to_float_modern(int i) {
    return std::bit_cast<float>(i);  // SAFE, constexpr, zero-overhead
}

```

`std::memcpy` is recognized by all compilers and optimized to the same code as `reinterpret_cast` dereference — but without the UB.

### The `char*` / `unsigned char*` / `std::byte*` Exception

These three types can alias **any** object type. This is how serialization, hashing, and network code legally inspect raw bytes:

```cpp

#include <cstddef>
#include <iostream>

void print_bytes(const void* ptr, std::size_t size) {
    // Legal: unsigned char* can alias anything
    const unsigned char* bytes = static_cast<const unsigned char*>(ptr);
    for (std::size_t i = 0; i < size; ++i) {
        std::printf("%02x ", bytes[i]);
    }
    std::puts("");
}

int main() {
    float f = 3.14f;
    print_bytes(&f, sizeof(f));  // Legal! Prints raw bytes of float

    int arr[] = {1, 2, 3};
    print_bytes(arr, sizeof(arr));  // Legal!

    // C++17 std::byte version:
    const std::byte* bp = reinterpret_cast<const std::byte*>(&f);
    // Legal to read through std::byte*
}

```

**Important:** The exception is **one-directional**. `char*` can alias `float`, but `float*` cannot alias `char[]`.

### `-fno-strict-aliasing`

This GCC/Clang flag **disables** the strict aliasing optimization:

```bash

g++ -O2 -fno-strict-aliasing code.cpp

```

**When it's acceptable:**

- Legacy C code that uses type-punning through unions  
- Code that interfaces with C libraries using `void*` casts
- Projects where the UB risk isn't worth the performance gain (e.g., some game engines)

**When it's NOT acceptable:**

- New modern C++ code — use `std::bit_cast` or `std::memcpy` instead
- Library code that must be portable across compilers

MSVC does **not** exploit strict aliasing by default (it behaves as if `-fno-strict-aliasing` is always on).

---

## Self-Assessment

### Q1: Show that accessing a float via an `int*` obtained by `reinterpret_cast` is UB under strict aliasing

```cpp

#include <iostream>
#include <cstring>
#include <bit>

int main() {
    float f = 3.14f;

    // ==== UB: strict aliasing violation ====
    int* bad_ptr = reinterpret_cast<int*>(&f);
    int bad_bits = *bad_ptr;  // UB! Reading float object through int*
    // The compiler may optimize away this read or return stale data

    std::cout << "UB result: " << bad_bits << "\n";  // May print anything

    // ==== CORRECT: use std::memcpy ====
    int good_bits;
    std::memcpy(&good_bits, &f, sizeof(good_bits));
    std::cout << "memcpy result: " << good_bits << "\n";  // 1078523331 (IEEE 754 bits of 3.14f)

    // ==== BEST (C++20): use std::bit_cast ====
    int best_bits = std::bit_cast<int>(f);
    std::cout << "bit_cast result: " << best_bits << "\n"; // 1078523331

    // Verify they agree (when UB happens to "work"):
    std::cout << "Match: " << (good_bits == best_bits) << "\n"; // 1

    return 0;
}

```

**How this works:**

- `reinterpret_cast<int*>(&f)` converts a `float*` to `int*` — the cast is legal but dereferencing is UB.
- The compiler assumes `int*` and `float*` never point to the same object, so it may:
  - Return a stale cached value
  - Reorder reads/writes past the aliasing access
  - Eliminate the read entirely
- `std::memcpy` copies bytes without type aliasing — always safe.
- `std::bit_cast<int>(f)` (C++20) is the ideal solution: safe, `constexpr`, zero-overhead.

### Q2: List the exceptions — `char*`, `unsigned char*`, and `std::byte*` can alias anything

The strict aliasing rule has these **exceptions** — accessing an object through these pointer types is always legal:

| Pointer type | Available since | Notes |
| --- | --- | --- |
| `char*` | C++98 | Can alias any type; used in legacy code |
| `unsigned char*` | C++98 | Preferred for raw byte access (no signedness issues) |
| `std::byte*` | C++17 | Strongest type — can only be manipulated via bitwise ops |
| `signed char*` | C++98 | Also aliases anything (signed/unsigned variant rule) |

```cpp

#include <cstddef>
#include <cstdint>
#include <iostream>

struct Packet {
    uint32_t header;
    uint16_t length;
    uint8_t  payload[64];
};

void inspect_memory(const Packet& pkt) {
    // ALL THREE of these are legal:

    // 1. char*
    const char* cp = reinterpret_cast<const char*>(&pkt);
    std::cout << "First byte (char): " << static_cast<int>(cp[0]) << "\n";

    // 2. unsigned char*
    const unsigned char* ucp = reinterpret_cast<const unsigned char*>(&pkt);
    std::cout << "First byte (uchar): " << static_cast<int>(ucp[0]) << "\n";

    // 3. std::byte* (C++17)
    const std::byte* bp = reinterpret_cast<const std::byte*>(&pkt);
    std::cout << "First byte (byte): " << std::to_integer<int>(bp[0]) << "\n";
}

int main() {
    Packet pkt{0x12345678, 10, {}};
    inspect_memory(pkt);  // Legal — no UB
    return 0;
}

```

**Why these exceptions exist:**

- Low-level operations (serialization, hashing, networking) need to inspect raw memory.
- `memcpy`, `memset`, and similar functions work through `void*`/`char*` internally.
- `std::byte` (C++17) adds type safety: you can't accidentally do arithmetic on byte pointers.

### Q3: Demonstrate that `-fno-strict-aliasing` disables this optimization and when that is acceptable

```cpp

#include <iostream>

// This function demonstrates the optimization opportunity
int check_aliasing(int* ip, float* fp) {
    *ip = 42;
    *fp = 3.14f;
    return *ip;  // With strict aliasing: compiler returns 42 (cached)
                  // Without: compiler reloads from memory
}

int main() {
    // Type-punning through union (common in C, UB in C++ strict aliasing)
    union Pun {
        float f;
        int i;
    };

    Pun p;
    p.f = 3.14f;

    // Reading p.i after writing p.f is UB in C++ (legal in C99)
    // With -fno-strict-aliasing, compilers treat this as defined behavior
    std::cout << "Union type pun: 0x" << std::hex << p.i << std::dec << "\n";

    // Compile and test:
    // g++ -O2 -fstrict-aliasing code.cpp -o strict     # May miscompile
    // g++ -O2 -fno-strict-aliasing code.cpp -o nostrict # Always "works"

    // Check what check_aliasing returns when pointers alias:
    Pun pun;
    int result = check_aliasing(&pun.i, &pun.f);
    std::cout << "check_aliasing result: " << result << "\n";
    // With strict aliasing: likely prints 42 (compiler caches)
    // Without: prints the int representation of 3.14f

    return 0;
}

```

**When `-fno-strict-aliasing` is acceptable:**

- ✅ Porting legacy C code that relies on union type-punning
- ✅ Interfacing with hardware/OS APIs that use `void*` casts extensively
- ✅ Game engines or multimedia code where the performance cost is negligible
- ✅ Projects already using it (e.g., Linux kernel compiles with `-fno-strict-aliasing`)

**When it's NOT acceptable:**

- ❌ New C++ library code — use `std::bit_cast` or `std::memcpy`
- ❌ Performance-critical numeric code — the optimizer may generate significantly worse code
- ❌ Code that must be portable to MSVC (which doesn't have an equivalent flag)

---

## Additional Examples

### Safe Type Punning with `std::bit_cast` (C++20)

```cpp

#include <bit>
#include <cstdint>
#include <iostream>
#include <iomanip>

int main() {
    // Float to int bits (IEEE 754 inspection)
    float f = -0.0f;
    uint32_t bits = std::bit_cast<uint32_t>(f);
    std::cout << "Bits of -0.0f: 0x" << std::hex << bits << "\n";  // 0x80000000

    // Double to uint64_t
    double d = 1.0;
    uint64_t dbits = std::bit_cast<uint64_t>(d);
    std::cout << "Bits of 1.0: 0x" << dbits << std::dec << "\n";   // 0x3ff0000000000000

    // Constexpr! (unlike memcpy)
    constexpr uint32_t pi_bits = std::bit_cast<uint32_t>(3.14159f);
    static_assert(pi_bits == 0x40490fd0 || pi_bits == 0x40490fdb);  // IEEE 754
}

```

### Network Byte Order Conversion (Legal Aliasing)

```cpp

#include <cstring>
#include <cstdint>
#include <iostream>

// Safe: uses memcpy, not reinterpret_cast dereference
uint32_t read_uint32_be(const unsigned char* buf) {
    uint32_t result;
    std::memcpy(&result, buf, sizeof(result));
    // Then convert from big-endian if needed
    return result;
}

int main() {
    unsigned char packet[] = {0x00, 0x00, 0x00, 0x42};
    uint32_t val = read_uint32_be(packet);
    std::cout << "Value: " << val << "\n";
}

```

---

## Notes

- **Never** dereference a `reinterpret_cast`-ed pointer of an unrelated type — use `std::bit_cast` or `std::memcpy`.
- `char*`, `unsigned char*`, and `std::byte*` can legally alias any type (but not the reverse).
- `-fno-strict-aliasing` is a compiler flag, not a language-level guarantee — code relying on it is not portable.
- `std::bit_cast` (C++20) is the modern, `constexpr`, zero-overhead solution for type punning.
- MSVC does not optimize based on strict aliasing, so bugs may only appear on GCC/Clang with `-O2`.
