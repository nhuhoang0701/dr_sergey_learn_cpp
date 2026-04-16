# Use Sanitizer coverage and libFuzzer for fuzz testing C++ code

**Category:** Tooling & Debugging  
**Item:** #249  
**Reference:** <https://llvm.org/docs/LibFuzzer.html>  

---

## Topic Overview

libFuzzer is an in-process, coverage-guided fuzzer. It generates random inputs to find bugs by maximizing code coverage.

```cpp

libFuzzer workflow:

  1. You write LLVMFuzzerTestOneInput(data, size)
  2. libFuzzer generates random inputs
  3. Coverage instrumentation guides mutation
  4. New code paths -> keep the input in corpus
  5. Crash detected -> save crashing input as artifact

  Mutations: bit flips, byte insertions, dictionary words,
             cross-over between corpus entries

```

---

## Self-Assessment

### Q1: Set up a fuzz target with `LLVMFuzzerTestOneInput`

```cpp

// fuzz_parser.cpp — fuzz target for a simple parser
#include <cstdint>
#include <cstddef>
#include <cstring>
#include <stdexcept>

// Function under test: parses a key=value string
bool parse_config(const char* input, size_t len) {
    if (len < 3) return false;

    // Find '=' separator
    const char* eq = static_cast<const char*>(std::memchr(input, '=', len));
    if (!eq) return false;

    size_t key_len = eq - input;
    size_t val_len = len - key_len - 1;

    // BUG: off-by-one when key_len == 0
    if (key_len == 0)
        return input[key_len - 1] == 'x';  // underflow!

    return key_len > 0 && val_len > 0;
}

// Fuzz target entry point:
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Call the function under test:
    parse_config(reinterpret_cast<const char*>(data), size);
    return 0;  // must return 0
}

```

```bash

# Compile with fuzzer and sanitizer:
$ clang++ -std=c++20 -g -O1 \
    -fsanitize=fuzzer,address \
    fuzz_parser.cpp -o fuzz_parser

# Run the fuzzer:
$ mkdir corpus
$ ./fuzz_parser corpus/
# INFO: Seed: 1234567
# INFO: Loaded 0 files from corpus/
# #2    INITED
# #128  NEW    cov: 12 ft: 13 corp: 3/15b
# #512  NEW    cov: 15 ft: 18 corp: 5/32b
# ==12345==ERROR: AddressSanitizer: stack-buffer-underflow
# READ of size 1 at 0x7ffd... (1 byte before buffer)
#   #0 parse_config(char const*, unsigned long) fuzz_parser.cpp:17
#
# artifact_prefix='./'; Test unit written to ./crash-abc123

# The fuzzer found input "=" (key_len==0) that triggers the bug

```

### Q2: Interpret crash artifacts and minimize

```bash

# View the crashing input:
$ xxd ./crash-abc123
# 00000000: 3d                    =
# The input is just "=" (a single equals sign)

# Reproduce the crash:
$ ./fuzz_parser ./crash-abc123
# ==ERROR: AddressSanitizer: stack-buffer-underflow
# Same crash, deterministic

# Minimize the crashing input:
$ ./fuzz_parser -minimize_crash=1 -exact_artifact_path=./minimized ./crash-abc123
# Tries progressively smaller inputs that still crash
# Output: minimized from 1 bytes to 1 byte (already minimal)

# For larger crashes, minimization removes irrelevant bytes:
# Original: "aaaa=bbbb\x00\xff\x01" (12 bytes)
# Minimized: "=" (1 byte) <- much easier to debug

# Merge corpus (remove redundant entries):
$ ./fuzz_parser -merge=1 corpus_merged/ corpus/
# Keeps only inputs that cover unique code paths

```

### Q3: Combine fuzzing with AddressSanitizer

```cpp

// fuzz_buffer.cpp — finds heap buffer overflow
#include <cstdint>
#include <cstddef>
#include <cstring>

void process_packet(const uint8_t* data, size_t size) {
    if (size < 4) return;

    // Read length from first 2 bytes
    uint16_t payload_len;
    std::memcpy(&payload_len, data, 2);

    // BUG: trusts user-controlled length without bounds check
    uint8_t* buffer = new uint8_t[64];
    if (payload_len > 0) {
        std::memcpy(buffer, data + 2, payload_len);  // heap overflow if payload_len > 62!
    }
    delete[] buffer;
}

extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    process_packet(data, size);
    return 0;
}

```

```bash

# Compile with ASan + fuzzer:
$ clang++ -std=c++20 -g -O1 \
    -fsanitize=fuzzer,address \
    fuzz_buffer.cpp -o fuzz_buffer

$ ./fuzz_buffer corpus_buffer/
# Within seconds:
# ==ERROR: AddressSanitizer: heap-buffer-overflow
# WRITE of size 200 at 0x60200000efb0
#   #0 process_packet fuzz_buffer.cpp:14
# artifact: ./crash-def456

# The fuzzer crafted input where first 2 bytes = large number,
# causing memcpy to overflow the 64-byte buffer

# Fix:
void process_packet_fixed(const uint8_t* data, size_t size) {
    if (size < 4) return;
    uint16_t payload_len;
    std::memcpy(&payload_len, data, 2);
    if (payload_len > 62 || payload_len + 2 > size) return;  // bounds check!
    uint8_t* buffer = new uint8_t[64];
    std::memcpy(buffer, data + 2, payload_len);
    delete[] buffer;
}

```

---

## Notes

- libFuzzer requires Clang; GCC does not support it natively.
- Always use ASan with fuzzing to catch memory bugs.
- Use `-max_total_time=60` to limit fuzzing duration in CI.
- Seed corpus (`corpus/` directory) with valid sample inputs for faster exploration.
- Use `-dict=protocol.dict` with protocol-specific tokens for better coverage.
