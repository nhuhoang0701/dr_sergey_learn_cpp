# Understand C++ exception table overhead and when to use -fno-exceptions

**Category:** Embedded & Constrained Systems  
**Standard:** C++17  
**Reference:** <https://itanium-cxx-abi.github.io/cxx-abi/abi-eh.html>  

---

## Topic Overview

### The Cost of C++ Exceptions

C++ exceptions use **zero-cost exception handling** (on Itanium ABI / GCC / Clang): there is no runtime overhead when exceptions are **not thrown**. However, enabling exceptions has significant **static costs** that matter on constrained systems:

| Cost type | Impact | Typical size |
| --- | --- | --- |
| **Unwind tables** (`.ARM.exidx`, `.eh_frame`) | Flash | 10–30% of `.text` size |
| **Landing pads** (catch clauses) | Flash | Per catch site |
| **Personality routine** | Flash | ~2–5 KB |
| **C++ runtime** (`__cxa_throw`, `__cxa_begin_catch`) | Flash | ~5–15 KB |
| **Stack usage** | RAM | Minimal (tables are in flash) |
| **Runtime cost** (when not throwing) | CPU | Zero on modern compilers |
| **Runtime cost** (when throwing) | CPU | Very slow (~1000× vs return) |

### Unwind Tables — Where the Bloat Comes From

For every function, the compiler emits a table entry describing how to unwind the stack frame (restore saved registers, run destructors). On ARM:

```cpp

.ARM.exidx section — one 8-byte entry per function
.ARM.extab section — extended unwind info for complex frames

```

For a firmware with 1,000 functions, that's at minimum 8 KB just for the index. Functions with local variables that have destructors need extended entries.

### Measuring the Impact

```bash

# Build with exceptions
arm-none-eabi-g++ -O2 -std=c++17 -o with_exc.elf main.cpp

# Build without exceptions
arm-none-eabi-g++ -O2 -std=c++17 -fno-exceptions -o no_exc.elf main.cpp

# Compare sizes
arm-none-eabi-size with_exc.elf no_exc.elf

# Show exception-related sections
arm-none-eabi-objdump -h with_exc.elf | grep -E "exidx|extab|eh_frame"

```

Typical results on a medium-sized project (~50 KB code):

| Build | .text | .ARM.exidx | .ARM.extab | Total flash |
| --- | --- | --- | --- | --- |
| With exceptions | 48 KB | 8 KB | 3 KB | 62 KB |
| Without exceptions | 45 KB | 0 | 0 | 48 KB |

That's a **23% reduction** from disabling exceptions.

### What `-fno-exceptions` Does

1. **Disables `throw` and `try`/`catch`** — using them is a compile error
2. **Removes unwind tables** — no `.ARM.exidx` / `.eh_frame` sections
3. **Changes `std::terminate` behavior** — throwing (e.g., from `std::optional::value()`) calls `std::abort()` directly
4. **Enables `operator new` to return null** — unless you link `--specs=nosys.specs` or use your own `operator new`

### What Breaks Without Exceptions

```cpp

// These are compile errors with -fno-exceptions:
throw std::runtime_error("oops");
try { f(); } catch (...) { }

// These call std::terminate() / abort():
std::optional<int> opt;
int x = opt.value();     // throws bad_optional_access → terminates

std::vector<int> v;
int y = v.at(10);        // throws out_of_range → terminates

auto p = std::make_unique<LargeObj>(); // new may return nullptr if not handled

```

### Coding Patterns Without Exceptions

```cpp

// Instead of exceptions — return error codes or std::expected
#include <cstdint>

enum class Error : uint8_t {
    None = 0,
    InvalidInput,
    Timeout,
    HardwareFault,
};

// Option 1: Error codes
struct Result {
    float value;
    Error error;
};

Result read_sensor() {
    if (!sensor_ready()) {
        return {0.0f, Error::Timeout};
    }
    float v = adc_read();
    if (v < 0.0f || v > 3.3f) {
        return {0.0f, Error::HardwareFault};
    }
    return {v, Error::None};
}

// Option 2: bool return with out parameter
bool parse_command(const char* input, Command& out) {
    if (!input) return false;
    // ... parse ...
    out = parsed;
    return true;
}

// Option 3: assert/abort for truly unrecoverable errors
void critical_init() {
    bool ok = configure_clocks();
    if (!ok) {
        // Log error and halt — no recovery possible
        error_log("Clock init failed");
        __disable_irq();
        while (true) {}
    }
}

```

### Providing `operator new` for `-fno-exceptions`

```cpp

#include <cstdlib>
#include <new>

// With -fno-exceptions, the default operator new calls std::terminate
// on failure. Provide your own that returns nullptr:
void* operator new(std::size_t size, const std::nothrow_t&) noexcept {
    return malloc(size);
}

void* operator new(std::size_t size) noexcept {
    void* p = malloc(size);
    if (!p) {
        // In embedded: log and halt rather than silently returning null
        error_handler("Out of memory");
    }
    return p;
}

void operator delete(void* p) noexcept { free(p); }
void operator delete(void* p, std::size_t) noexcept { free(p); }

```

### `-fno-rtti` — the Companion Flag

RTTI (Run-Time Type Information) adds:

- `type_info` objects for every polymorphic class (~50–100 bytes each)
- `typeid` and `dynamic_cast` support

```bash

arm-none-eabi-g++ -fno-exceptions -fno-rtti -o minimal.elf main.cpp

```

| Feature | Disabled by | Saves |
| --- | --- | --- |
| `throw`/`catch` | `-fno-exceptions` | 15–30% flash |
| `dynamic_cast`, `typeid` | `-fno-rtti` | 2–5% flash |
| Both | Both flags | 20–35% flash |

### CMake Integration

```cmake

target_compile_options(firmware PRIVATE
    -fno-exceptions
    -fno-rtti
    -fno-unwind-tables       # Explicitly remove unwind tables
    -fno-asynchronous-unwind-tables
)

# GCC link-time flag to remove exception handling library
target_link_options(firmware PRIVATE
    -fno-exceptions
    --specs=nano.specs       # Use newlib-nano (smaller libc)
)

```

---

## Self-Assessment

### Q1: Why does zero-cost exception handling still cost flash

"Zero-cost" means no runtime overhead **when exceptions are not thrown** — no extra instructions in the normal code path. But the compiler must still emit **unwind tables** that describe how to restore each stack frame during unwinding. These tables are stored in flash (`.ARM.exidx`, `.ARM.extab`, or `.eh_frame`) and can be 10–30% of the text section size. On a 512 KB MCU, that could be 50–100 KB of flash wasted on data that's never used if you never throw.

### Q2: What's the main risk of using `-fno-exceptions`

Standard library functions that normally throw exceptions will call `std::terminate()` instead, causing an **immediate abort**. This includes:

- `std::vector::at()`, `std::optional::value()`, `std::get<>()` on `std::variant`
- `operator new` on allocation failure
- Any third-party library that uses exceptions internally

You must audit all code paths and replace exception-relying APIs with non-throwing alternatives (e.g., `operator[]` instead of `at()`, `std::get_if` instead of `std::get`).

### Q3: Show the CMake configuration for a minimal-overhead C++ embedded build

```cmake

add_executable(firmware main.cpp startup.cpp)

target_compile_features(firmware PRIVATE cxx_std_17)

target_compile_options(firmware PRIVATE
    -Os                     # Optimize for size
    -fno-exceptions         # No exception support
    -fno-rtti               # No RTTI
    -fno-unwind-tables
    -fno-asynchronous-unwind-tables
    -ffunction-sections     # Put each function in its own section
    -fdata-sections         # Put each variable in its own section
    -Wall -Wextra
)

target_link_options(firmware PRIVATE
    -Wl,--gc-sections       # Remove unused sections
    -Wl,--print-memory-usage
    --specs=nano.specs       # Tiny C library
    --specs=nosys.specs      # No system calls
    -T${CMAKE_SOURCE_DIR}/linker.ld  # Custom linker script
)

```

---

## Notes

- Always measure: build with and without exceptions and compare `arm-none-eabi-size` output
- `-ffunction-sections` + `-Wl,--gc-sections` is essential — removes unused code
- Some vendors (e.g., Arduino, Mbed) default to `-fno-exceptions` for their platforms
- If you need exception-like error propagation, use `std::expected` (C++23) or a lightweight Result type
- Clang and GCC behave identically with these flags
