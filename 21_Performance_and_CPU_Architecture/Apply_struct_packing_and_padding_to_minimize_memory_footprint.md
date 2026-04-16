# Apply struct packing and padding to minimize memory footprint

**Category:** Performance & CPU Architecture  
**Item:** #630  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/object#Alignment>  

---

## Topic Overview

Compilers insert padding between struct members to satisfy alignment requirements. Field ordering dramatically affects struct size.

```cpp

Poorly ordered (24 bytes):       Well ordered (16 bytes):

struct Bad {                     struct Good {
  char  a;   // offset 0          double d;  // offset 0  (8)
  // 7 bytes padding               int    i;  // offset 8  (4)
  double d;  // offset 8          short  s;  // offset 12 (2)
  int   i;   // offset 16         char   a;  // offset 14 (1)
  // 4 bytes padding               char   b;  // offset 15 (1)
};           // size = 24         };           // size = 16

```

| Rule | Description |
| --- | --- |
| Alignment | Each member aligned to its `alignof` value |
| Padding | Compiler inserts bytes to satisfy alignment |
| Struct size | Multiple of largest member's alignment |
| Reorder trick | Sort fields largest-to-smallest |
| `#pragma pack(1)` | Remove padding (may cause misaligned access) |

---

## Self-Assessment

### Q1: Reveal padding with `sizeof` and `offsetof`, then reorder

```cpp

#include <iostream>
#include <cstddef>

// BAD layout: lots of padding
struct Padded {
    char  a;     // 1 byte + 7 padding
    double d;    // 8 bytes
    char  b;     // 1 byte + 3 padding
    int   i;     // 4 bytes
};               // Total: 24 bytes (8 bytes wasted!)

// GOOD layout: minimal padding
struct Compact {
    double d;    // 8 bytes (offset 0)
    int    i;    // 4 bytes (offset 8)
    char   a;    // 1 byte  (offset 12)
    char   b;    // 1 byte  (offset 13)
};               // Total: 16 bytes (0 bytes wasted)

int main() {
    // Reveal padding in Padded:
    std::cout << "=== Padded (bad order) ===\n";
    std::cout << "sizeof:    " << sizeof(Padded) << '\n';           // 24
    std::cout << "a offset:  " << offsetof(Padded, a) << '\n';     // 0
    std::cout << "d offset:  " << offsetof(Padded, d) << '\n';     // 8  (7 padding!)
    std::cout << "b offset:  " << offsetof(Padded, b) << '\n';     // 16
    std::cout << "i offset:  " << offsetof(Padded, i) << '\n';     // 20 (3 padding!)

    std::cout << "\n=== Compact (good order) ===\n";
    std::cout << "sizeof:    " << sizeof(Compact) << '\n';          // 16
    std::cout << "d offset:  " << offsetof(Compact, d) << '\n';    // 0
    std::cout << "i offset:  " << offsetof(Compact, i) << '\n';    // 8
    std::cout << "a offset:  " << offsetof(Compact, a) << '\n';    // 12
    std::cout << "b offset:  " << offsetof(Compact, b) << '\n';    // 13

    // For an array of 1M elements:
    // Padded:  24 MB
    // Compact: 16 MB  (33% less memory, better cache utilization)
}

```

### Q2: `__attribute__((packed))` and alignment fault risk

```cpp

#include <iostream>
#include <cstddef>
#include <cstdint>

// Force no padding:
struct __attribute__((packed)) PackedStruct {
    char  a;     // offset 0
    double d;    // offset 1  (MISALIGNED!)
    int   i;     // offset 9  (MISALIGNED!)
    char  b;     // offset 13
};               // Total: 14 bytes (vs 24 unpacked)

struct NormalStruct {
    char  a;
    double d;
    int   i;
    char  b;
};

int main() {
    std::cout << "Packed:  " << sizeof(PackedStruct) << '\n';   // 14
    std::cout << "Normal:  " << sizeof(NormalStruct) << '\n';   // 24
    std::cout << "d offset (packed): " << offsetof(PackedStruct, d) << '\n'; // 1

    // RISKS of packing:
    // 1. x86/x64: misaligned access works but is SLOWER
    //    (hardware handles it, ~2x penalty for crossing cache line)
    //
    // 2. ARM/MIPS/RISC-V (strict alignment):
    //    SIGBUS or hardware trap on misaligned access!
    //
    // 3. Cannot take address of packed member safely:
    PackedStruct ps{.a='x', .d=3.14, .i=42, .b='y'};
    // double* ptr = &ps.d;  // WARNING: taking address of packed member
    //                       // ptr may be misaligned!

    // BETTER approach: reorder fields instead of packing
    struct Reordered {
        double d;  // 8
        int    i;  // 4
        char   a;  // 1
        char   b;  // 1 + 2 padding
    };              // 16 bytes, properly aligned

    std::cout << "Reordered: " << sizeof(Reordered) << '\n';  // 16
}

```

### Q3: Packed vs aligned struct cache efficiency in arrays

```cpp

#include <iostream>
#include <vector>
#include <chrono>
#include <numeric>

struct Aligned {
    double value;  // 8
    int    id;     // 4
    char   flag;   // 1 + 3 padding
};                 // 16 bytes, naturally aligned

struct __attribute__((packed)) Packed {
    double value;  // 8
    int    id;     // 4
    char   flag;   // 1
};                 // 13 bytes, misaligned doubles in array!

int main() {
    constexpr int N = 1'000'000;

    // Arrays:
    std::vector<Aligned> aligned(N);
    std::vector<Packed> packed(N);

    for (int i = 0; i < N; ++i) {
        aligned[i] = {static_cast<double>(i), i, 'a'};
        packed[i]  = {static_cast<double>(i), i, 'a'};
    }

    // Benchmark: sum all values
    auto bench = [](auto& vec, const char* label) {
        auto t0 = std::chrono::high_resolution_clock::now();
        double sum = 0;
        for (int rep = 0; rep < 100; ++rep) {
            for (auto& e : vec) sum += e.value;
        }
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0);
        std::cout << label << ": " << us.count() << "us (sum=" << sum << ")\n";
    };

    bench(aligned, "Aligned (16B)");
    bench(packed,  "Packed  (13B)");

    // Typical result on x86:
    // Aligned (16B): ~150ms  -- doubles always cache-line aligned
    // Packed  (13B): ~180ms  -- some doubles cross cache lines
    //
    // Memory: aligned=16MB, packed=13MB (19% less)
    // Speed: aligned ~20% faster (no misalignment penalty)
    //
    // Verdict: reorder fields > pack. Only pack for wire protocols.
}

```

---

## Notes

- **Rule of thumb:** sort fields from largest `alignof` to smallest.
- `#pragma pack(push, 1)` / `#pragma pack(pop)` for MSVC compatibility.
- Use `static_assert(sizeof(T) == expected)` to catch layout changes.
- `-Wpadded` (GCC/Clang) warns about implicit padding.
- Packing is appropriate for network protocols and file formats where layout must be exact.
- For arrays, the savings compound: 8 bytes × 1M elements = 8MB less memory.
