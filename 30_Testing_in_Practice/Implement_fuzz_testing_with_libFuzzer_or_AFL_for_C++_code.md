# Implement fuzz testing with libFuzzer or AFL for C++ code

**Category:** Testing in Practice

---

## Topic Overview

**Fuzz testing** feeds random/mutated inputs to code to find crashes, hangs, and memory errors. It discovers edge cases humans never think of. Combined with sanitizers, it catches bugs like buffer overflows, use-after-free, and undefined behavior automatically.

### Fuzzer Comparison

| Feature | libFuzzer | AFL++ | Honggfuzz |
| --- | --- | --- | --- |
| **Type** | In-process, coverage-guided | Fork-based, coverage-guided | Fork/in-process |
| **Speed** | Very fast (no fork) | Fast with persistent mode | Fast |
| **Integration** | Built into Clang | Separate tool | Separate tool |
| **Corpus management** | Built-in | Built-in | Built-in |
| **Sanitizer support** | Native (ASan, UBSan, MSan) | Via compiler flags | Via compiler flags |
| **Dictionaries** | Yes | Yes | Yes |
| **Structure-aware** | Custom mutators | Custom mutators | Custom mutators |

---

## Self-Assessment

### Q1: Write a libFuzzer fuzz target for a parser

**Answer:**

```cpp

// === fuzz_json_parser.cpp ===
// Fuzz target: entry point called by libFuzzer with random data

#include <cstdint>
#include <cstddef>
#include <string>
#include <string_view>
#include <stdexcept>

// Our JSON-like parser under test
struct JsonValue {
    enum Type { Null, Bool, Number, String, Array, Object };
    Type type;
    std::string string_val;
    double number_val;
};

JsonValue parse_json(std::string_view input);  // Function under test

// === libFuzzer entry point ===
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    // Convert raw bytes to string_view
    std::string_view input(reinterpret_cast<const char*>(data), size);

    try {
        auto result = parse_json(input);
        // Optional: verify invariants on successful parse
        // e.g., serialized result parses back to same value
    } catch (const std::exception&) {
        // Expected: invalid inputs should throw, not crash
    }
    // Any crash, ASAN error, or UBSan error is a bug
    return 0;
}

```

```bash

# === Build with libFuzzer + sanitizers ===
clang++ -g -O1 \
    -fsanitize=fuzzer,address,undefined \
    -fno-omit-frame-pointer \
    fuzz_json_parser.cpp json_parser.cpp \
    -o fuzz_json

# === Run ===
mkdir -p corpus
./fuzz_json corpus/ \
    -max_len=4096 \
    -timeout=10 \
    -dict=json.dict \
    -jobs=4 \
    -workers=4

# === Optional seed corpus ===
echo '{"key": "value"}' > corpus/seed1
echo '[1, 2, 3]' > corpus/seed2
echo 'null' > corpus/seed3

```

```cpp

# === json.dict (fuzzer dictionary) ===
"{"
"}"
"["
"]"
":"
","
"true"
"false"
"null"
"\""

```

### Q2: Build a CMake project with fuzzing targets

**Answer:**

```cmake

# === CMakeLists.txt ===
cmake_minimum_required(VERSION 3.16)
project(my_project)

option(ENABLE_FUZZING "Build fuzz targets" OFF)

add_library(my_parser src/parser.cpp)
target_include_directories(my_parser PUBLIC include/)

if(ENABLE_FUZZING)
    # Require Clang for libFuzzer
    if(NOT CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        message(FATAL_ERROR "Fuzzing requires Clang")
    endif()

    # Common fuzzing flags
    set(FUZZ_FLAGS "-fsanitize=fuzzer,address,undefined -fno-omit-frame-pointer")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${FUZZ_FLAGS}")
    set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} ${FUZZ_FLAGS}")

    # Add fuzz targets
    add_executable(fuzz_parser fuzz/fuzz_parser.cpp)
    target_link_libraries(fuzz_parser my_parser)

    add_executable(fuzz_serializer fuzz/fuzz_serializer.cpp)
    target_link_libraries(fuzz_serializer my_parser)

    # Regression test: run corpus as unit tests
    file(GLOB PARSER_CORPUS fuzz/corpus/parser/*)
    foreach(testcase ${PARSER_CORPUS})
        get_filename_component(name ${testcase} NAME)
        add_test(
            NAME fuzz_regression_${name}
            COMMAND fuzz_parser ${testcase}
        )
    endforeach()
endif()

```

```bash

# Build and run
cmake -B build-fuzz -DENABLE_FUZZING=ON \
    -DCMAKE_CXX_COMPILER=clang++
cmake --build build-fuzz

# Run fuzzer
./build-fuzz/fuzz_parser fuzz/corpus/parser/ -max_total_time=300

# When a crash is found, it's saved as crash-<hash>
# Add it to corpus for regression testing
cp crash-* fuzz/corpus/parser/

```

### Q3: Write a structure-aware fuzzer with custom mutators

**Answer:**

```cpp

// === fuzz_protocol.cpp ===
// Structure-aware fuzzer for a binary protocol

#include <cstdint>
#include <cstddef>
#include <cstring>
#include <algorithm>

// Protocol header
struct __attribute__((packed)) Header {
    uint8_t  version;
    uint8_t  msg_type;
    uint16_t payload_length;
    uint32_t checksum;
};

void process_message(const uint8_t* data, size_t size);  // Under test

// Basic fuzz target
extern "C" int LLVMFuzzerTestOneInput(const uint8_t* data, size_t size) {
    if (size < sizeof(Header)) return 0;  // Need at least a header

    process_message(data, size);
    return 0;
}

// === Custom mutator: generates valid protocol structures ===
extern "C" size_t LLVMFuzzerCustomMutator(
    uint8_t* data, size_t size, size_t max_size, unsigned int seed) {

    // Ensure we have at least a header
    if (max_size < sizeof(Header)) return 0;

    // Sometimes mutate header fields intelligently
    std::minstd_rand rng(seed);

    if (size >= sizeof(Header)) {
        Header* hdr = reinterpret_cast<Header*>(data);

        switch (rng() % 5) {
            case 0: hdr->version = rng() % 4; break;        // Valid versions: 0-3
            case 1: hdr->msg_type = rng() % 10; break;      // Valid types: 0-9
            case 2: {
                // Fix payload_length to match actual payload
                uint16_t payload = size - sizeof(Header);
                hdr->payload_length = payload;
                break;
            }
            case 3: {
                // Corrupt checksum (should be caught by validation)
                hdr->checksum ^= rng();
                break;
            }
            default:
                // Let default mutation handle it
                break;
        }
    }

    // Use libFuzzer's default mutation on the payload portion
    return size;
}

// === Custom cross-over for combining two inputs ===
extern "C" size_t LLVMFuzzerCustomCrossOver(
    const uint8_t* data1, size_t size1,
    const uint8_t* data2, size_t size2,
    uint8_t* out, size_t max_out_size,
    unsigned int seed) {

    // Take header from one, payload from another
    if (size1 < sizeof(Header) || size2 < sizeof(Header))
        return 0;

    size_t payload2_len = size2 - sizeof(Header);
    size_t total = sizeof(Header) + payload2_len;
    if (total > max_out_size) return 0;

    // Header from input 1
    std::memcpy(out, data1, sizeof(Header));
    // Payload from input 2
    std::memcpy(out + sizeof(Header), data2 + sizeof(Header), payload2_len);

    // Fix length field
    auto* hdr = reinterpret_cast<Header*>(out);
    hdr->payload_length = static_cast<uint16_t>(payload2_len);

    return total;
}

```

```yaml

# === CI integration: OSS-Fuzz style ===
name: Fuzz Testing
on:
  schedule:

    - cron: '0 2 * * *'  # Nightly

jobs:
  fuzz:
    runs-on: ubuntu-latest
    steps:

      - uses: actions/checkout@v4

      - name: Build fuzz targets

        run: |
          cmake -B build-fuzz -DENABLE_FUZZING=ON \
              -DCMAKE_CXX_COMPILER=clang++
          cmake --build build-fuzz

      - name: Run fuzzer (time-limited)

        run: |
          ./build-fuzz/fuzz_parser fuzz/corpus/parser/ \
              -max_total_time=600 \
              -print_final_stats=1

      - name: Upload new corpus

        uses: actions/upload-artifact@v4
        with:
          name: fuzz-corpus
          path: fuzz/corpus/

      - name: Upload crashes

        if: failure()
        uses: actions/upload-artifact@v4
        with:
          name: fuzz-crashes
          path: crash-*

```

---

## Notes

- **Always fuzz with sanitizers** (ASan + UBSan minimum) — the fuzzer finds inputs, sanitizers detect the bugs
- Save crash-reproducing inputs as regression tests in your corpus directory
- Use dictionaries (`.dict` files) to seed the fuzzer with protocol-specific tokens
- `max_len` should match realistic input sizes — too large wastes time, too small misses bugs
- Structure-aware fuzzing (custom mutators) is dramatically more effective for protocols and file formats
- Coverage-guided fuzzing finds bugs in minutes that random testing misses in hours
- Consider OSS-Fuzz for open-source projects — Google runs it continuously for free
- libFuzzer is built into Clang; no separate installation needed
