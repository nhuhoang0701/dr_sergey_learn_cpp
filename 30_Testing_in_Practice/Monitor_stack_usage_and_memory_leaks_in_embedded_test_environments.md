# Monitor stack usage and memory leaks in embedded test environments

**Category:** Testing in Practice

---

## Topic Overview

Stack overflow and memory leaks are critical in embedded systems where RAM is limited (often 8-256 KB). Unlike desktop apps, embedded systems can't recover from stack overflow — it silently corrupts data. Monitoring techniques: **compile-time stack analysis**, **runtime stack painting**, **leak detection in tests**, and **linker map file analysis**.

### Memory Monitoring Approaches

| Technique | Stack | Heap | When | Overhead |
| --- | --- | --- | --- | --- |
| **`-fstack-usage` (GCC)** | Per-function analysis | No | Compile time | Zero |
| **Stack painting** | High-water mark | No | Runtime | Low |
| **ASan (host)** | Overflow detect | Leaks | Test time | 2x memory |
| **Valgrind/Memcheck** | No | Full leak report | Test time | 10-30x slower |
| **Linker map** | Total allocation | Static/BSS | Link time | Zero |
| **Custom new/delete** | No | Track heap | Runtime | Low |

---

## Self-Assessment

### Q1: Analyze stack usage at compile time and link time

**Answer:**

```bash

# === GCC -fstack-usage: per-function stack analysis ===
arm-none-eabi-g++ -fstack-usage -c src/main.cpp -o build/main.o
# Creates main.su alongside main.o:
# main.cpp:15:6:main  128  static
# main.cpp:30:6:process_data  256  dynamic,bounded
# main.cpp:45:6:handle_isr  32  static

# Parse .su files to find stack hogs
find build -name '*.su' -exec cat {} \; | sort -t: -k5 -n -r | head -20

```

```python

# === stack_analyzer.py: Parse .su files and check against limits ===
import sys
from pathlib import Path

def analyze_stack(build_dir: str, max_stack: int = 512):
    violations = []
    for su_file in Path(build_dir).rglob('*.su'):
        for line in su_file.read_text().splitlines():
            parts = line.split('\t')
            if len(parts) >= 2:
                func = parts[0]
                stack_bytes = int(parts[1])
                if stack_bytes > max_stack:
                    violations.append((func, stack_bytes))

    if violations:
        print(f"Stack violations (limit: {max_stack} bytes):")
        for func, size in sorted(violations, key=lambda x: -x[1]):
            print(f"  {func}: {size} bytes")
        return False
    return True

if __name__ == '__main__':
    ok = analyze_stack('build', max_stack=512)
    sys.exit(0 if ok else 1)

```

```cmake

# === CMake: Enable stack analysis and linker map ===
target_compile_options(firmware PRIVATE
    -fstack-usage
    -Wstack-usage=512     # Warn if any function uses >512 bytes
)

target_link_options(firmware PRIVATE
    -Wl,-Map=${CMAKE_BINARY_DIR}/firmware.map
    -Wl,--print-memory-usage    # Show RAM/Flash usage summary
)

# Post-build: check stack limits
add_custom_command(TARGET firmware POST_BUILD
    COMMAND python3 ${CMAKE_SOURCE_DIR}/tools/stack_analyzer.py
            ${CMAKE_BINARY_DIR} --max-stack 512
    COMMENT "Checking stack usage..."
)

```

### Q2: Runtime stack monitoring with stack painting

**Answer:**

```cpp

// === Stack painting: fill stack with known pattern at boot ===
// Implemented in startup code or RTOS task creation

#include <cstdint>
#include <cstring>
#include <algorithm>

constexpr uint32_t STACK_PAINT_PATTERN = 0xDEADBEEF;

void paint_stack(uint32_t* stack_bottom, size_t size_words) {
    // Fill unused stack with pattern
    // Called at boot before main (or at task creation in RTOS)
    for (size_t i = 0; i < size_words; ++i)
        stack_bottom[i] = STACK_PAINT_PATTERN;
}

size_t measure_stack_used(const uint32_t* stack_bottom, size_t size_words) {
    // Count from bottom up to find first unpainted word
    size_t unused = 0;
    for (size_t i = 0; i < size_words; ++i) {
        if (stack_bottom[i] == STACK_PAINT_PATTERN)
            unused++;
        else
            break;
    }
    return (size_words - unused) * sizeof(uint32_t);
}

float stack_usage_percent(const uint32_t* stack_bottom, size_t size_words) {
    size_t used = measure_stack_used(stack_bottom, size_words);
    return (static_cast<float>(used) / (size_words * sizeof(uint32_t))) * 100.0f;
}

// === Host test for the stack analysis functions ===
#include <gtest/gtest.h>
#include <vector>

TEST(StackPaintTest, MeasuresUsedStack) {
    // Simulate a 1KB stack, fully painted
    std::vector<uint32_t> stack(256, STACK_PAINT_PATTERN);

    // Simulate usage: overwrite top 64 words
    for (size_t i = 192; i < 256; ++i)
        stack[i] = 0x12345678;

    size_t used = measure_stack_used(stack.data(), stack.size());
    EXPECT_EQ(used, 64 * sizeof(uint32_t));  // 256 bytes used
}

TEST(StackPaintTest, EmptyStackShowsZeroUsage) {
    std::vector<uint32_t> stack(256, STACK_PAINT_PATTERN);
    EXPECT_EQ(measure_stack_used(stack.data(), stack.size()), 0);
}

TEST(StackPaintTest, FullStackShowsFullUsage) {
    std::vector<uint32_t> stack(256, 0x12345678);  // All overwritten
    size_t used = measure_stack_used(stack.data(), stack.size());
    EXPECT_EQ(used, 256 * sizeof(uint32_t));
}

```

### Q3: Integrate memory leak detection into test infrastructure

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <cstdlib>
#include <atomic>
#include <iostream>

// === Global new/delete override for leak tracking ===
namespace {
    std::atomic<int64_t> g_alloc_count{0};
    std::atomic<int64_t> g_alloc_bytes{0};
    std::atomic<int64_t> g_peak_bytes{0};
}

void* operator new(size_t size) {
    void* ptr = std::malloc(size);
    if (!ptr) throw std::bad_alloc();
    g_alloc_count.fetch_add(1, std::memory_order_relaxed);
    g_alloc_bytes.fetch_add(size, std::memory_order_relaxed);
    auto current = g_alloc_bytes.load(std::memory_order_relaxed);
    auto peak = g_peak_bytes.load(std::memory_order_relaxed);
    while (current > peak &&
           !g_peak_bytes.compare_exchange_weak(peak, current,
               std::memory_order_relaxed)) {}
    return ptr;
}

void operator delete(void* ptr) noexcept {
    if (ptr) {
        g_alloc_count.fetch_sub(1, std::memory_order_relaxed);
        std::free(ptr);
    }
}

void operator delete(void* ptr, size_t) noexcept {
    ::operator delete(ptr);
}

// === Test listener that reports memory stats ===
class MemoryCheckListener : public ::testing::EmptyTestEventListener {
public:
    void OnTestStart(const ::testing::TestInfo&) override {
        start_count_ = g_alloc_count.load();
    }

    void OnTestEnd(const ::testing::TestInfo& info) override {
        auto end_count = g_alloc_count.load();
        auto leaked = end_count - start_count_;
        if (leaked > 0) {
            std::cerr << "[LEAK] " << info.test_suite_name() << "."
                      << info.name() << ": "
                      << leaked << " allocations leaked\n";
        }
    }

    void OnTestProgramEnd(const ::testing::UnitTest&) override {
        std::cout << "Peak heap usage: "
                  << g_peak_bytes.load() << " bytes\n";
    }

private:
    int64_t start_count_ = 0;
};

// Register in main:
// auto& listeners = ::testing::UnitTest::GetInstance()->listeners();
// listeners.Append(new MemoryCheckListener);

```

---

## Notes

- `-fstack-usage` is GCC-specific; Clang has limited support — use `-Wframe-larger-than=` for warnings
- Stack painting works on any platform but only detects high-water mark, not per-call depth
- Global `operator new/delete` override is a simple but effective leak detector for test builds
- Parse linker `.map` files to verify total static RAM usage fits in the MCU's SRAM
- RTOS task stacks should have 20-30% margin above measured high-water mark
- Combine compile-time (`-fstack-usage`) and runtime (painting) for defense in depth
- In CI: fail the build if any function exceeds the stack budget or if leaks are detected
- `-Wl,--print-memory-usage` gives a quick summary: `FLASH: 45KB / 256KB, RAM: 12KB / 64KB`
