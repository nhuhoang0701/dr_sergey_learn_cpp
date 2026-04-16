# Use Protocol Buffers and FlatBuffers for Efficient Network Serialization

**Category:** Networking & I/O  
**Standard:** C++17/20  
**Reference:** [Protobuf C++](https://protobuf.dev/reference/cpp/), [FlatBuffers Guide](https://flatbuffers.dev/flatbuffers_guide_use_cpp.html)  

---

## Topic Overview

Serialization is the bottleneck between application logic and the network. **Protocol Buffers** (protobuf) and **FlatBuffers** are Google's two dominant schema-based serialization formats for C++. Protobuf is the industry standard for RPC (gRPC) and configuration, while FlatBuffers targets latency-sensitive use cases (games, real-time systems) where zero-copy deserialization eliminates parsing overhead entirely.

The key architectural difference: protobuf **decodes** a binary wire format into C++ objects (allocating memory, copying strings), while FlatBuffers **reads directly from the buffer** — the serialized bytes *are* the data structure. This means FlatBuffers deserialization is O(1) — you get a pointer and start reading — while protobuf is O(n) in message size.

| Aspect                 | Protocol Buffers              | FlatBuffers                   |
| --- | --- | --- |
| Deserialization        | O(n), full parse + allocation | O(1), zero-copy access        |
| Serialization          | O(n), fast                    | O(n), slightly slower (builder) |
| Memory usage           | Dynamic allocation            | Single contiguous buffer      |
| Random field access    | Yes (parsed objects)          | Yes (vtable offsets)          |
| Schema evolution       | Excellent (field numbers)     | Good (field IDs, deprecation) |
| Human-readable format  | TextFormat, JSON              | JSON (via flatc)              |
| RPC integration        | gRPC native                   | gRPC plugin available         |
| Mutable after parse    | Yes                           | No (read-only by default)     |
| Code generation        | `protoc`                      | `flatc`                       |

```cpp

Protobuf Wire Format:              FlatBuffers Memory Layout:
┌──────┬──────┬──────┐             ┌──────────┬──────────┬────────────┐
│field1│field2│field3│  ← encoded  │  vtable  │  table   │  strings/  │
│tag+  │tag+  │tag+  │    bytes    │(offsets) │(scalars +│  vectors   │
│value │value │value │             │          │ offsets) │  (inline)  │
└──────┴──────┴──────┘             └──────────┴──────────┴────────────┘
        ↓ parse                              ↓ zero-copy
   C++ objects with                    Direct pointer access
   heap allocations                    into buffer bytes

```

Schema versioning is where protobuf excels: field numbers are stable across versions, new fields have defaults, and unknown fields are preserved during round-trips. FlatBuffers uses field IDs similarly but does not preserve unknown fields. Both formats are **not self-describing** — you need the schema to decode, unlike JSON.

---

## Self-Assessment

### Q1: Define equivalent protobuf and FlatBuffers schemas and show serialization/deserialization in C++

```protobuf

// sensor_data.proto
syntax = "proto3";
package sensor;

message SensorReading {
    uint64 timestamp_ns = 1;
    uint32 sensor_id    = 2;
    float  temperature  = 3;
    float  humidity     = 4;
    repeated float accelerometer = 5;  // [x, y, z]
    Status status = 6;

    enum Status {
        OK      = 0;
        WARNING = 1;
        ERROR   = 2;
    }
}

message SensorBatch {
    repeated SensorReading readings = 1;
    string source_node = 2;
}

```

```flatbuffers

// sensor_data.fbs
namespace sensor;

enum Status : byte { Ok = 0, Warning = 1, Error = 2 }

table SensorReading {
    timestamp_ns: uint64;
    sensor_id:    uint32;
    temperature:  float;
    humidity:     float;
    accelerometer: [float];  // [x, y, z]
    status:       Status = Ok;
}

table SensorBatch {
    readings:    [SensorReading];
    source_node: string;
}

root_type SensorBatch;

```

```cpp

// protobuf_usage.cpp
#include "sensor_data.pb.h"
#include <string>
#include <chrono>
#include <iostream>

void protobuf_roundtrip() {
    // --- Serialize ---
    sensor::SensorBatch batch;
    batch.set_source_node("node-42");

    auto* reading = batch.add_readings();
    reading->set_timestamp_ns(
        std::chrono::steady_clock::now().time_since_epoch().count());
    reading->set_sensor_id(1);
    reading->set_temperature(23.5f);
    reading->set_humidity(45.0f);
    reading->add_accelerometer(0.01f);
    reading->add_accelerometer(-9.81f);
    reading->add_accelerometer(0.02f);
    reading->set_status(sensor::SensorReading::OK);

    // Serialize to bytes
    std::string wire_data;
    batch.SerializeToString(&wire_data);
    std::cout << "Protobuf size: " << wire_data.size() << " bytes\n";

    // --- Deserialize --- (O(n) parse)
    sensor::SensorBatch parsed;
    if (!parsed.ParseFromString(wire_data)) {
        std::cerr << "Protobuf parse failed\n";
        return;
    }

    std::cout << "Source: " << parsed.source_node()
              << ", readings: " << parsed.readings_size()
              << ", temp: " << parsed.readings(0).temperature() << "\n";
}

```

```cpp

// flatbuffers_usage.cpp
#include "sensor_data_generated.h"  // flatc output
#include <flatbuffers/flatbuffers.h>
#include <iostream>
#include <chrono>

void flatbuffers_roundtrip() {
    // --- Serialize ---
    flatbuffers::FlatBufferBuilder builder(256);

    // Build inner data first (FlatBuffers builds bottom-up)
    std::vector<float> accel = {0.01f, -9.81f, 0.02f};
    auto accel_vec = builder.CreateVector(accel);

    auto reading = sensor::CreateSensorReading(
        builder,
        std::chrono::steady_clock::now().time_since_epoch().count(),
        1,       // sensor_id
        23.5f,   // temperature
        45.0f,   // humidity
        accel_vec,
        sensor::Status_Ok
    );

    std::vector<flatbuffers::Offset<sensor::SensorReading>> readings_vec;
    readings_vec.push_back(reading);

    auto batch = sensor::CreateSensorBatch(
        builder,
        builder.CreateVector(readings_vec),
        builder.CreateString("node-42")
    );
    builder.Finish(batch);

    // Wire data is builder.GetBufferPointer(), size builder.GetSize()
    uint8_t* buf = builder.GetBufferPointer();
    size_t   sz  = builder.GetSize();
    std::cout << "FlatBuffers size: " << sz << " bytes\n";

    // --- Deserialize --- (O(1) zero-copy)
    auto parsed = sensor::GetSensorBatch(buf);

    // Direct access — NO parsing, NO allocation
    std::cout << "Source: " << parsed->source_node()->str()
              << ", readings: " << parsed->readings()->size()
              << ", temp: " << parsed->readings()->Get(0)->temperature()
              << "\n";
}

```

### Q2: How do you handle schema evolution (adding/removing fields) safely in both formats

```protobuf

// sensor_data_v2.proto — evolved schema
syntax = "proto3";
package sensor;

message SensorReading {
    uint64 timestamp_ns    = 1;
    uint32 sensor_id       = 2;
    float  temperature     = 3;
    float  humidity        = 4;
    repeated float accelerometer = 5;
    Status status          = 6;

    // NEW in v2 — safe, old clients ignore this field
    string firmware_version = 7;

    // DEPRECATED — keep the field number reserved
    // float old_field = 8;  // removed
    reserved 8;
    reserved "old_field";

    enum Status {
        OK      = 0;
        WARNING = 1;
        ERROR   = 2;
        MAINTENANCE = 3;  // NEW enum value — safe
    }
}

```

```cpp

// Schema evolution rules — critical reference

/*

 * PROTOBUF EVOLUTION RULES:
 * ✅ Add new fields with new field numbers
 * ✅ Add new enum values
 * ✅ Remove fields (reserve the number!)
 * ✅ Rename fields (wire format uses numbers, not names)
 * ❌ Change field number of existing field
 * ❌ Change field type (int32 → string)
 * ❌ Change repeated ↔ scalar

 *

 * FLATBUFFERS EVOLUTION RULES:
 * ✅ Add new fields at END of table
 * ✅ Deprecate fields (id preserved, accessor removed)
 * ✅ Add new enum values
 * ❌ Remove fields (deprecate instead)
 * ❌ Reorder fields
 * ❌ Change field types
 * ❌ Add fields to structs (structs are fixed-size)

 */

// Forward compatibility example: old code reading new data
void read_forward_compatible(const std::string& wire_data) {
    sensor::SensorReading old_reader;
    // ParseFromString succeeds — unknown field 7 is silently preserved
    old_reader.ParseFromString(wire_data);
    std::cout << "temp: " << old_reader.temperature() << "\n";
    // firmware_version field is in UnknownFieldSet — not lost

    // Re-serialize preserves unknown fields (protobuf 3.5+)
    std::string re_serialized;
    old_reader.SerializeToString(&re_serialized);
    // Field 7 data is still present in re_serialized
}

```

### Q3: Compare serialization performance and show how to benchmark protobuf vs FlatBuffers

```cpp

#include <chrono>
#include <iostream>
#include <vector>
#include <numeric>
#include <cstring>

// Benchmark harness
template<typename Fn>
double bench_ns(Fn&& fn, int iterations) {
    // Warmup
    for (int i = 0; i < 100; ++i) fn();

    auto start = std::chrono::high_resolution_clock::now();
    for (int i = 0; i < iterations; ++i) fn();
    auto end = std::chrono::high_resolution_clock::now();

    return std::chrono::duration_cast<std::chrono::nanoseconds>(
        end - start).count() / static_cast<double>(iterations);
}

void run_benchmarks() {
    constexpr int N = 100'000;

    // --- Protobuf serialize ---
    double pb_ser = bench_ns([&] {
        sensor::SensorBatch batch;
        batch.set_source_node("node-42");
        auto* r = batch.add_readings();
        r->set_timestamp_ns(1234567890);
        r->set_sensor_id(1);
        r->set_temperature(23.5f);
        r->set_humidity(45.0f);
        r->add_accelerometer(0.01f);
        r->add_accelerometer(-9.81f);
        r->add_accelerometer(0.02f);
        std::string out;
        batch.SerializeToString(&out);
    }, N);

    // --- FlatBuffers serialize ---
    flatbuffers::FlatBufferBuilder builder(256);
    double fb_ser = bench_ns([&] {
        builder.Clear();
        std::vector<float> accel = {0.01f, -9.81f, 0.02f};
        auto av = builder.CreateVector(accel);
        auto r = sensor::CreateSensorReading(
            builder, 1234567890, 1, 23.5f, 45.0f, av, sensor::Status_Ok);
        std::vector<flatbuffers::Offset<sensor::SensorReading>> rv;
        rv.push_back(r);
        auto batch = sensor::CreateSensorBatch(
            builder, builder.CreateVector(rv),
            builder.CreateString("node-42"));
        builder.Finish(batch);
    }, N);

    // --- Protobuf deserialize + access ---
    sensor::SensorBatch pb;
    pb.set_source_node("node-42");
    auto* pr = pb.add_readings();
    pr->set_timestamp_ns(1234567890);
    pr->set_sensor_id(1);
    pr->set_temperature(23.5f);
    std::string wire;
    pb.SerializeToString(&wire);

    double pb_de = bench_ns([&] {
        sensor::SensorBatch parsed;
        parsed.ParseFromString(wire);
        volatile float t = parsed.readings(0).temperature(); (void)t;
    }, N);

    // --- FlatBuffers deserialize + access (zero-copy) ---
    builder.Clear();
    auto av = builder.CreateVector(std::vector<float>{0.01f, -9.81f, 0.02f});
    auto r = sensor::CreateSensorReading(
        builder, 1234567890, 1, 23.5f, 45.0f, av, sensor::Status_Ok);
    std::vector<flatbuffers::Offset<sensor::SensorReading>> rv{r};
    auto batch = sensor::CreateSensorBatch(
        builder, builder.CreateVector(rv), builder.CreateString("node-42"));
    builder.Finish(batch);

    auto* fb_buf = builder.GetBufferPointer();
    double fb_de = bench_ns([&] {
        auto parsed = sensor::GetSensorBatch(fb_buf);
        volatile float t = parsed->readings()->Get(0)->temperature(); (void)t;
    }, N);

    std::cout << "=== Benchmark Results (ns/op, N=" << N << ") ===\n"
              << "                Serialize    Deserialize\n"
              << "Protobuf:       " << pb_ser << "        " << pb_de << "\n"
              << "FlatBuffers:    " << fb_ser << "        " << fb_de << "\n";

    // Typical results:
    // Protobuf:      ~300-800 ns     ~200-500 ns
    // FlatBuffers:   ~400-1000 ns    ~5-20 ns  (zero-copy!)
}

```

---

## Notes

- FlatBuffers deserialization is **O(1)** and allocation-free — this matters enormously for high-frequency data (100K+ messages/sec).
- Protobuf's `Arena` allocator (`google::protobuf::Arena`) dramatically reduces allocation overhead by bulk-allocating and bulk-freeing.
- Always use `reserved` in protobuf when removing fields — prevents accidental reuse of field numbers, which causes data corruption.
- FlatBuffers `Verifier` should be used on untrusted input: `flatbuffers::Verifier v(buf, size); assert(VerifySensorBatchBuffer(v));`.
- FlatBuffers tables are read-only after creation; use `UnPackTo()` if you need in-place mutation (generates "object API" with `--gen-object-api`).
- Protobuf `SerializeToString` can fail silently if required fields (proto2) are unset; always check the return value.
- Neither format supports random-access writes — if you need incremental updates, use Cap'n Proto.
