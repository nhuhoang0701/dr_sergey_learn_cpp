# Design data serialization architecture with protobuf, flatbuffers, or msgpack

**Category:** Project Architecture

---

## Topic Overview

**Serialization** converts in-memory objects to bytes for storage or transmission, and deserialization converts them back. Choosing the right format depends on performance, schema evolution, language interop, and human readability. In C++ projects, protobuf, FlatBuffers, and msgpack cover most production use cases.

### Serialization Formats Comparison

| Format | Schema | Zero-copy | Size | Speed | Schema Evolution |
| --- | --- | --- | --- | --- | --- |
| **Protobuf** | Required (.proto) | No | Small | Fast | Excellent (field numbers) |
| **FlatBuffers** | Required (.fbs) | Yes | Small-Medium | Fastest | Good |
| **msgpack** | Optional | No | Smallest | Fast | Manual |
| **JSON** | Optional | No | Large | Slow | Fragile |
| **Cap'n Proto** | Required | Yes | Small | Fastest | Good |

---

## Self-Assessment

### Q1: Design a serialization abstraction layer

**Answer:**

```cpp

#include <vector>
#include <string>
#include <cstdint>
#include <optional>
#include <span>

// === Domain types (serialization-agnostic) ===
struct SensorReading {
    uint32_t sensor_id;
    double temperature;
    double humidity;
    uint64_t timestamp_ms;
    std::string location;
    std::vector<double> raw_samples;
};

// === Serializer interface ===
class ISerializer {
public:
    virtual ~ISerializer() = default;
    virtual std::vector<uint8_t> serialize(
        const SensorReading& reading) const = 0;
    virtual std::optional<SensorReading> deserialize(
        std::span<const uint8_t> data) const = 0;
    virtual const char* format_name() const = 0;
};

// === Protobuf implementation ===
// sensor.proto:
// syntax = "proto3";
// message SensorReading {
//   uint32 sensor_id = 1;
//   double temperature = 2;
//   double humidity = 3;
//   uint64 timestamp_ms = 4;
//   string location = 5;
//   repeated double raw_samples = 6;
// }

class ProtobufSerializer : public ISerializer {
public:
    std::vector<uint8_t> serialize(
            const SensorReading& reading) const override {
        proto::SensorReading msg;
        msg.set_sensor_id(reading.sensor_id);
        msg.set_temperature(reading.temperature);
        msg.set_humidity(reading.humidity);
        msg.set_timestamp_ms(reading.timestamp_ms);
        msg.set_location(reading.location);
        for (double s : reading.raw_samples)
            msg.add_raw_samples(s);

        std::vector<uint8_t> buffer(msg.ByteSizeLong());
        msg.SerializeToArray(buffer.data(), buffer.size());
        return buffer;
    }

    std::optional<SensorReading> deserialize(
            std::span<const uint8_t> data) const override {
        proto::SensorReading msg;
        if (!msg.ParseFromArray(data.data(), data.size()))
            return std::nullopt;

        return SensorReading{
            msg.sensor_id(), msg.temperature(), msg.humidity(),
            msg.timestamp_ms(), msg.location(),
            {msg.raw_samples().begin(), msg.raw_samples().end()}
        };
    }

    const char* format_name() const override { return "protobuf"; }
};

```

### Q2: FlatBuffers zero-copy implementation

**Answer:**

```cpp

// === sensor.fbs ===
// namespace Sensor;
// table Reading {
//   sensor_id: uint32;
//   temperature: double;
//   humidity: double;
//   timestamp_ms: uint64;
//   location: string;
//   raw_samples: [double];
// }
// root_type Reading;

#include "sensor_generated.h"  // flatc-generated

class FlatBufferSerializer : public ISerializer {
public:
    std::vector<uint8_t> serialize(
            const SensorReading& reading) const override {
        flatbuffers::FlatBufferBuilder builder(256);

        auto location = builder.CreateString(reading.location);
        auto samples = builder.CreateVector(reading.raw_samples);

        auto fb = Sensor::CreateReading(builder,
            reading.sensor_id,
            reading.temperature,
            reading.humidity,
            reading.timestamp_ms,
            location,
            samples);

        builder.Finish(fb);
        auto* buf = builder.GetBufferPointer();
        return {buf, buf + builder.GetSize()};
    }

    std::optional<SensorReading> deserialize(
            std::span<const uint8_t> data) const override {
        // Zero-copy: directly access buffer without parsing!
        auto* fb = Sensor::GetReading(data.data());
        if (!fb) return std::nullopt;

        SensorReading result;
        result.sensor_id = fb->sensor_id();
        result.temperature = fb->temperature();
        result.humidity = fb->humidity();
        result.timestamp_ms = fb->timestamp_ms();
        if (fb->location())
            result.location = fb->location()->str();
        if (fb->raw_samples())
            result.raw_samples.assign(
                fb->raw_samples()->begin(),
                fb->raw_samples()->end());
        return result;
    }

    // Zero-copy access without full deserialization:
    static double get_temperature(std::span<const uint8_t> data) {
        // No allocation, no parsing — direct memory read
        return Sensor::GetReading(data.data())->temperature();
    }

    const char* format_name() const override { return "flatbuffers"; }
};

```

### Q3: Handle schema evolution and versioning

**Answer:**

```cpp

// === Schema evolution rules ===

// Protobuf: SAFE changes
// - Add new fields (with new field numbers)
// - Remove fields (old readers ignore unknown fields)
// - Rename fields (wire format uses numbers, not names)

// Protobuf: BREAKING changes
// - Change field number
// - Change field type (int32 -> string)
// - Remove field number and reuse it

// === Versioned serialization architecture ===
struct MessageHeader {
    uint8_t version;       // Schema version
    uint8_t format;        // 0=protobuf, 1=flatbuf, 2=msgpack
    uint16_t message_type; // Application-defined type ID
    uint32_t payload_size; // Payload length in bytes
};
static_assert(sizeof(MessageHeader) == 8);

class VersionedSerializer {
public:
    std::vector<uint8_t> serialize(uint16_t type,
                                    const std::vector<uint8_t>& payload) {
        MessageHeader header{
            .version = CURRENT_VERSION,
            .format = static_cast<uint8_t>(format_),
            .message_type = type,
            .payload_size = static_cast<uint32_t>(payload.size())
        };

        std::vector<uint8_t> result(sizeof(header) + payload.size());
        std::memcpy(result.data(), &header, sizeof(header));
        std::memcpy(result.data() + sizeof(header),
                    payload.data(), payload.size());
        return result;
    }

    // Deserialize with version handling
    template<typename T>
    std::optional<T> deserialize(std::span<const uint8_t> data,
                                  ISerializer& serializer) {
        if (data.size() < sizeof(MessageHeader))
            return std::nullopt;

        MessageHeader header;
        std::memcpy(&header, data.data(), sizeof(header));

        if (header.version > CURRENT_VERSION) {
            // Future version: try to read compatible subset
            // Protobuf/FlatBuffers handle this gracefully
        }

        auto payload = data.subspan(sizeof(header), header.payload_size);
        return serializer.deserialize(payload);
    }

    static constexpr uint8_t CURRENT_VERSION = 3;
    int format_ = 0;
};

```

---

## Notes

- **Protobuf** is the default choice: excellent schema evolution, wide language support, reasonable performance
- **FlatBuffers** for latency-critical paths: zero-copy access avoids deserialization entirely
- **msgpack** for simple cases: no schema files needed, self-describing, very compact
- **Never use raw struct memcpy for serialization**: endianness, padding, alignment are platform-dependent
- Always version your wire format — header with version byte enables graceful migration
- Serialization layer should be in infrastructure, not domain: domain types should never `#include <protobuf>`
- For IPC within the same process/machine, FlatBuffers or shared memory avoids serialization entirely
