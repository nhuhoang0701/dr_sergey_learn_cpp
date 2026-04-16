# Use std::spanstream and memory buffers for in-memory serialization

**Category:** Serialization & Data Formats  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/io/basic_ispanstream>  

---

## Topic Overview

C++23 introduces `std::spanstream` — a stream that operates on a fixed-size external buffer. Unlike `std::stringstream`, it performs **zero allocations** and works directly on a pre-allocated memory region.

### spanstream vs stringstream

| Feature | stringstream | spanstream (C++23) |
| --- | --- | --- |
| Buffer | Internal, grows as needed | External, fixed size |
| Allocation | Yes (heap) | None |
| Use case | General purpose | Pre-allocated buffers, embedded |
| Performance | Good | Better (no allocation) |

### Basic Usage

```cpp

#include <spanstream>
#include <span>
#include <array>
#include <iostream>
#include <cstdint>

void spanstream_demo() {
    // Fixed buffer — no heap allocation
    std::array<char, 256> buffer{};

    // Write to buffer
    std::ospanstream out(buffer);
    out << "Temperature: " << 23.5 << " C, "
        << "Pressure: " << 1013.25 << " hPa";

    // Read what was written
    auto written = out.span();
    std::cout << "Written " << written.size() << " bytes: "
              << std::string_view(written.data(), written.size()) << "\n";

    // Read from buffer
    std::ispanstream in(written);
    std::string label;
    double temp;
    in >> label >> temp;
    std::cout << "Parsed: " << label << " " << temp << "\n";
}

```

### Binary Serialization with spanstream

```cpp

#include <spanstream>
#include <cstdint>
#include <array>
#include <cstring>

struct SensorReading {
    uint32_t sensor_id;
    float value;
    uint64_t timestamp;
};

void binary_serialize() {
    std::array<char, 1024> buffer{};
    std::ospanstream out(buffer);

    // Write binary data
    SensorReading r{42, 23.5f, 1700000000ULL};
    out.write(reinterpret_cast<const char*>(&r), sizeof(r));
    out.write(reinterpret_cast<const char*>(&r), sizeof(r));

    auto written = out.span();

    // Read back
    std::ispanstream in(written);
    SensorReading r1, r2;
    in.read(reinterpret_cast<char*>(&r1), sizeof(r1));
    in.read(reinterpret_cast<char*>(&r2), sizeof(r2));
}

```

### Use with Pre-Allocated Memory (Embedded/Real-Time)

```cpp

#include <spanstream>
#include <span>
#include <cstdint>
#include <iostream>

// In embedded/real-time: buffer is statically allocated
alignas(64) static char telemetry_buffer[4096];

void format_telemetry(uint32_t seq, double lat, double lon, float alt) {
    std::ospanstream out(std::span{telemetry_buffer});

    out << seq << "," << lat << "," << lon << "," << alt;

    auto data = out.span();
    // Send data.data() / data.size() over radio link
    // Zero heap allocation!
}

```

---

## Self-Assessment

### Q1: Show how spanstream avoids allocation compared to stringstream

```cpp

#include <spanstream>
#include <sstream>
#include <array>
#include <iostream>
#include <chrono>

void compare() {
    // stringstream: allocates internally
    {
        std::stringstream ss;
        ss << "Hello " << 42 << " " << 3.14;
        auto str = ss.str(); // Another allocation for the copy
    }

    // spanstream: zero allocation
    {
        std::array<char, 64> buf{};
        std::ospanstream os(buf);
        os << "Hello " << 42 << " " << 3.14;
        auto span = os.span(); // No copy — view into buf
        // span.data() points directly into buf
    }
}

```

### Q2: Use spanstream for formatting log messages without allocation

```cpp

#include <spanstream>
#include <array>
#include <string_view>

enum class LogLevel { INFO, WARN, ERROR };

std::string_view format_log(std::span<char> buf,
                             LogLevel level,
                             std::string_view msg,
                             int code) {
    std::ospanstream os(buf);
    os << (level == LogLevel::INFO ? "INFO" :
           level == LogLevel::WARN ? "WARN" : "ERROR")
       << " [" << code << "] " << msg;
    auto result = os.span();
    return {result.data(), static_cast<size_t>(result.size())};
}

void example() {
    std::array<char, 256> buf;
    auto msg = format_log(buf, LogLevel::ERROR, "Timeout", 504);
    // msg = "ERROR [504] Timeout" — no allocation
}

```

### Q3: When should you still use stringstream over spanstream

Use `stringstream` when you don't know the output size in advance and need the buffer to grow dynamically. `spanstream` requires a pre-allocated fixed buffer — if the output exceeds the buffer, the stream enters a fail state. For most formatting tasks where the size is bounded, `spanstream` is superior.

---

## Notes

- `std::spanstream` is in `<spanstream>` (C++23) — check compiler support.
- The buffer is NOT null-terminated by default — use `span().size()` for the written length.
- Ideal for embedded, real-time, and high-performance logging where allocation is forbidden.
- For pre-C++23: use `std::ostrstream` (deprecated) or write directly to a `char[]` with `snprintf`.
