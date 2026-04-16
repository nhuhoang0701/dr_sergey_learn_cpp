# Use MessagePack and CBOR for compact binary serialization

**Category:** Serialization & Data Formats  
**Standard:** C++17  
**Reference:** <https://msgpack.org/> · <https://cbor.io/>  

---

## Topic Overview

MessagePack and CBOR are self-describing binary serialization formats that are more compact than JSON while remaining schema-free. They're ideal for logging, caching, and inter-service communication.

### Comparison

| Feature | JSON | MessagePack | CBOR |
| --- | --- | --- | --- |
| Format | Text | Binary | Binary |
| Self-describing | Yes | Yes | Yes |
| Size vs JSON | Baseline | ~50-70% smaller | ~50-70% smaller |
| Standardized | RFC 8259 | msgpack spec | RFC 8949 |
| Streaming | Limited | Yes | Yes |
| Extensions | No | Yes (ext types) | Yes (tags) |

### Using nlohmann/json for MessagePack and CBOR

```cpp

#include <nlohmann/json.hpp>
#include <iostream>
#include <vector>
#include <iomanip>

using json = nlohmann::json;

int main() {
    json data = {
        {"name", "Alice"},
        {"age", 30},
        {"scores", {95, 87, 92}},
        {"active", true}
    };

    // JSON (text)
    auto json_str = data.dump();
    std::cout << "JSON size: " << json_str.size() << " bytes\n";
    // ~55 bytes

    // MessagePack (binary)
    auto msgpack = json::to_msgpack(data);
    std::cout << "MessagePack size: " << msgpack.size() << " bytes\n";
    // ~35 bytes

    // CBOR (binary)
    auto cbor = json::to_cbor(data);
    std::cout << "CBOR size: " << cbor.size() << " bytes\n";
    // ~36 bytes

    // Deserialize MessagePack
    auto restored = json::from_msgpack(msgpack);
    std::cout << "\nRestored: " << restored.dump(2) << "\n";

    // Hex dump of MessagePack
    std::cout << "\nMessagePack hex: ";
    for (auto b : msgpack) {
        std::cout << std::hex << std::setw(2) << std::setfill('0')
                  << static_cast<int>(b) << " ";
    }
    std::cout << "\n";
}

```

### msgpack-c Library (High Performance)

```cpp

#include <msgpack.hpp>
#include <iostream>
#include <string>
#include <vector>
#include <sstream>

struct Sensor {
    std::string id;
    double value;
    uint64_t timestamp;
    MSGPACK_DEFINE(id, value, timestamp);
};

void msgpack_c_example() {
    // Serialize
    Sensor s{"temp-01", 23.5, 1700000000};
    msgpack::sbuffer buf;
    msgpack::pack(buf, s);
    std::cout << "Packed size: " << buf.size() << " bytes\n";

    // Deserialize
    auto handle = msgpack::unpack(buf.data(), buf.size());
    Sensor restored;
    handle.get().convert(restored);
    std::cout << "Sensor: " << restored.id
              << " value=" << restored.value << "\n";

    // Streaming: pack multiple objects into one buffer
    msgpack::sbuffer stream;
    for (int i = 0; i < 5; ++i) {
        Sensor si{"sensor-" + std::to_string(i), i * 1.1, 0};
        msgpack::pack(stream, si);
    }
    std::cout << "Stream size (5 sensors): " << stream.size() << " bytes\n";
}

```

---

## Self-Assessment

### Q1: Show the size advantage of MessagePack over JSON for a typical API response

```cpp

#include <nlohmann/json.hpp>
#include <iostream>

int main() {
    // Typical API response
    nlohmann::json response;
    for (int i = 0; i < 100; ++i) {
        response.push_back({
            {"id", i},
            {"name", "User " + std::to_string(i)},
            {"email", "user" + std::to_string(i) + "@example.com"},
            {"score", 75.5 + i},
            {"active", i % 3 != 0}
        });
    }

    auto json_bytes = response.dump().size();
    auto msgpack_bytes = nlohmann::json::to_msgpack(response).size();
    auto cbor_bytes = nlohmann::json::to_cbor(response).size();

    std::cout << "JSON:        " << json_bytes << " bytes\n";
    std::cout << "MessagePack: " << msgpack_bytes << " bytes ("
              << (100 - 100 * msgpack_bytes / json_bytes) << "% smaller)\n";
    std::cout << "CBOR:        " << cbor_bytes << " bytes ("
              << (100 - 100 * cbor_bytes / json_bytes) << "% smaller)\n";
}

```

### Q2: When to choose MessagePack vs CBOR

- **MessagePack**: simpler spec, more language bindings, widely used in Redis/Fluentd/MessagePack-RPC
- **CBOR**: IETF standard (RFC 8949), supports tags for dates/bignum/etc., used in COSE/FIDO2/WebAuthn
- Both are roughly equivalent in size and performance

### Q3: Handle streaming serialization of time-series data

```cpp

// Write 1M sensor readings into a single buffer efficiently
#include <msgpack.hpp>
#include <chrono>

void stream_timeseries(const char* filename) {
    msgpack::sbuffer buf;
    for (int i = 0; i < 1'000'000; ++i) {
        msgpack::packer<msgpack::sbuffer> pk(&buf);
        pk.pack_array(3);
        pk.pack(i);           // sequence
        pk.pack(23.5 + i * 0.001); // value
        pk.pack(1700000000ULL + i); // timestamp
    }
    // Write buf to file — readers can unpack sequentially
    // No need to load entire file into memory
}

```

---

## Notes

- For nlohmann/json, MessagePack and CBOR support is built-in — no extra dependencies.
- For maximum performance, use the dedicated msgpack-c library with `MSGPACK_DEFINE` macros.
- CBOR tags (tag 0 = date string, tag 1 = epoch) provide standardized type extensions.
- Both formats handle nested objects, arrays, null, and binary data natively.
