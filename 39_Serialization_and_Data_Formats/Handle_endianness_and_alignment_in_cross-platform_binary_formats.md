# Handle endianness and alignment in cross-platform binary formats

**Category:** Serialization & Data Formats  
**Standard:** C++20/23  
**Reference:** <https://en.cppreference.com/w/cpp/types/endian>  

---

## Topic Overview

Binary data exchanged between different architectures must account for **endianness** (byte order) and **alignment** (padding between struct members). Ignoring these produces silent data corruption.

### Endianness Detection (C++20)

```cpp

#include <bit>
#include <cstdint>
#include <iostream>

void detect_endianness() {
    if constexpr (std::endian::native == std::endian::little) {
        std::cout << "Little-endian (x86, ARM default)\n";
    } else if constexpr (std::endian::native == std::endian::big) {
        std::cout << "Big-endian (network byte order, some POWER)\n";
    } else {
        std::cout << "Mixed-endian (rare)\n";
    }
}

```

### Byte Swapping (C++23)

```cpp

#include <bit>
#include <cstdint>
#include <iostream>

// C++23: std::byteswap
void swap_example() {
    uint32_t host_val = 0x01020304;
    uint32_t swapped = std::byteswap(host_val);
    std::cout << std::hex;
    std::cout << "Host:    0x" << host_val << "\n";   // 0x01020304
    std::cout << "Swapped: 0x" << swapped << "\n";     // 0x04030201
}

// Network-to-host conversion
template<typename T>
T from_network_order(T val) {
    if constexpr (std::endian::native == std::endian::little) {
        return std::byteswap(val);
    } else {
        return val;  // Already big-endian
    }
}

template<typename T>
T to_network_order(T val) {
    return from_network_order(val);  // Symmetric operation
}

```

### Alignment and Padding

```cpp

#include <cstdint>
#include <iostream>
#include <cstddef>

// Struct padding differs between compilers and architectures!
struct Unpacked {
    uint8_t  a;    // offset 0
    // 3 bytes padding
    uint32_t b;    // offset 4
    uint16_t c;    // offset 8
    // 2 bytes padding
};
// sizeof = 12 (with padding)

// Packed struct — no padding, but unaligned access may be slow/UB
#pragma pack(push, 1)
struct Packed {
    uint8_t  a;    // offset 0
    uint32_t b;    // offset 1 (unaligned!)
    uint16_t c;    // offset 5
};
#pragma pack(pop)
// sizeof = 7

// Safe approach: serialize field by field
void serialize_safe(const Unpacked& s, uint8_t* buf) {
    size_t pos = 0;
    std::memcpy(buf + pos, &s.a, 1); pos += 1;
    uint32_t b_net = to_network_order(s.b);
    std::memcpy(buf + pos, &b_net, 4); pos += 4;
    uint16_t c_net = to_network_order(s.c);
    std::memcpy(buf + pos, &c_net, 2); pos += 2;
    // Always 7 bytes, regardless of compiler padding
}

int main() {
    std::cout << "Unpacked size: " << sizeof(Unpacked) << "\n"; // 12
    std::cout << "Packed size:   " << sizeof(Packed) << "\n";   // 7
    std::cout << "a offset: " << offsetof(Unpacked, a) << "\n"; // 0
    std::cout << "b offset: " << offsetof(Unpacked, b) << "\n"; // 4
    std::cout << "c offset: " << offsetof(Unpacked, c) << "\n"; // 8
}

```

---

## Self-Assessment

### Q1: Write a portable binary writer that always produces big-endian output

```cpp

#include <bit>
#include <cstdint>
#include <vector>
#include <cstring>

class PortableBinaryWriter {
    std::vector<uint8_t> buf_;
public:
    void write_u8(uint8_t v)   { buf_.push_back(v); }
    void write_u16(uint16_t v) { write_be(v); }
    void write_u32(uint32_t v) { write_be(v); }
    void write_u64(uint64_t v) { write_be(v); }

    void write_string(const std::string& s) {
        write_u32(static_cast<uint32_t>(s.size()));
        buf_.insert(buf_.end(), s.begin(), s.end());
    }

    const std::vector<uint8_t>& data() const { return buf_; }

private:
    template<typename T>
    void write_be(T val) {
        if constexpr (std::endian::native == std::endian::little) {
            val = std::byteswap(val);
        }
        auto p = reinterpret_cast<const uint8_t*>(&val);
        buf_.insert(buf_.end(), p, p + sizeof(T));
    }
};

```

### Q2: Explain why `#pragma pack(1)` is dangerous for performance

Packed structs force unaligned memory access. On x86, this works but is slower (extra microops). On ARM and some RISC architectures, unaligned access can cause a **hardware exception** (bus error / SIGBUS). Even on x86, unaligned atomics are never guaranteed to work correctly. The safe approach is to use `memcpy` for field-by-field serialization.

### Q3: Handle mixed-endian data from a legacy protocol

```cpp

#include <cstdint>
#include <cstring>
#include <bit>

// Some legacy protocols mix endianness (e.g., PDP-11 "middle-endian")
// Handle this by explicit per-field byte order specification

struct LegacyPacket {
    uint16_t type;   // big-endian
    uint32_t length; // little-endian (legacy decision)
    uint16_t flags;  // big-endian
};

LegacyPacket parse_legacy(const uint8_t* buf) {
    LegacyPacket pkt;
    size_t pos = 0;

    std::memcpy(&pkt.type, buf + pos, 2); pos += 2;
    // type is big-endian on wire
    if constexpr (std::endian::native == std::endian::little)
        pkt.type = std::byteswap(pkt.type);

    std::memcpy(&pkt.length, buf + pos, 4); pos += 4;
    // length is little-endian on wire
    if constexpr (std::endian::native == std::endian::big)
        pkt.length = std::byteswap(pkt.length);

    std::memcpy(&pkt.flags, buf + pos, 2); pos += 2;
    if constexpr (std::endian::native == std::endian::little)
        pkt.flags = std::byteswap(pkt.flags);

    return pkt;
}

```

---

## Notes

- **Always use `memcpy`** for reading multi-byte values from buffers — never `reinterpret_cast`.
- C++23 `std::byteswap` replaces platform-specific `__builtin_bswap32` / `_byteswap_ulong`.
- Network protocols conventionally use big-endian ("network byte order").
- Modern protocols (FlatBuffers, Cap'n Proto) use little-endian to avoid swaps on x86/ARM.
- `#pragma pack` is non-standard — use field-by-field serialization for portability.
