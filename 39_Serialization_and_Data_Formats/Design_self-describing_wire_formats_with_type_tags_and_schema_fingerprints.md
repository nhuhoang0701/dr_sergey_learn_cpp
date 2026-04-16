# Design self-describing wire formats with type tags and schema fingerprints

**Category:** Serialization & Data Formats  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Type%E2%80%93length%E2%80%93value>  

---

## Topic Overview

A **self-describing** wire format carries enough metadata for a reader to interpret the data without prior schema agreement. Type tags identify field types, and schema fingerprints allow fast compatibility checks.

### Type Tag System

```cpp

#include <cstdint>
#include <string>
#include <variant>
#include <vector>
#include <iostream>
#include <cstring>

enum class WireType : uint8_t {
    Uint32   = 0,
    Int32    = 1,
    Float64  = 2,
    String   = 3,
    Bytes    = 4,
    Bool     = 5,
    Array    = 6,  // Followed by element WireType + count
    Null     = 7,
};

// Each field on the wire:
//   [1 byte] WireType
//   [2 bytes] field_id
//   [4 bytes] length (for variable-length types)
//   [N bytes] data

struct FieldHeader {
    WireType  type;
    uint16_t  field_id;
    uint32_t  data_length;  // 0 for fixed-size types
};

class SelfDescribingWriter {
    std::vector<uint8_t> buf_;
public:
    void write_uint32(uint16_t field_id, uint32_t val) {
        FieldHeader h{WireType::Uint32, field_id, sizeof(val)};
        append(h);
        append(val);
    }

    void write_string(uint16_t field_id, const std::string& s) {
        FieldHeader h{WireType::String, field_id, static_cast<uint32_t>(s.size())};
        append(h);
        buf_.insert(buf_.end(), s.begin(), s.end());
    }

    void write_float64(uint16_t field_id, double val) {
        FieldHeader h{WireType::Float64, field_id, sizeof(val)};
        append(h);
        append(val);
    }

    const std::vector<uint8_t>& buffer() const { return buf_; }

private:
    template<typename T>
    void append(const T& v) {
        auto p = reinterpret_cast<const uint8_t*>(&v);
        buf_.insert(buf_.end(), p, p + sizeof(T));
    }
};

class SelfDescribingReader {
    const uint8_t* data_;
    size_t size_, pos_ = 0;
public:
    SelfDescribingReader(const uint8_t* data, size_t size)
        : data_(data), size_(size) {}

    bool has_next() const { return pos_ + sizeof(FieldHeader) <= size_; }

    void read_next() {
        FieldHeader h;
        std::memcpy(&h, data_ + pos_, sizeof(h));
        pos_ += sizeof(h);

        std::cout << "Field " << h.field_id << " [";
        switch (h.type) {
            case WireType::Uint32: {
                uint32_t val;
                std::memcpy(&val, data_ + pos_, sizeof(val));
                std::cout << "uint32]: " << val;
                break;
            }
            case WireType::String: {
                std::string s(reinterpret_cast<const char*>(data_ + pos_), h.data_length);
                std::cout << "string]: \"" << s << "\"";
                break;
            }
            case WireType::Float64: {
                double val;
                std::memcpy(&val, data_ + pos_, sizeof(val));
                std::cout << "float64]: " << val;
                break;
            }
            default:
                std::cout << "unknown]: skipping " << h.data_length << " bytes";
        }
        std::cout << "\n";
        pos_ += h.data_length;
    }
};

int main() {
    SelfDescribingWriter w;
    w.write_string(1, "Alice");
    w.write_uint32(2, 30);
    w.write_float64(3, 98.6);

    SelfDescribingReader r(w.buffer().data(), w.buffer().size());
    while (r.has_next()) r.read_next();
    // Output:
    //   Field 1 [string]: "Alice"
    //   Field 2 [uint32]: 30
    //   Field 3 [float64]: 98.6
}

```

---

## Self-Assessment

### Q1: Compare self-describing formats vs schema-based formats

| Aspect | Self-Describing | Schema-Based |
| --- | --- | --- |
| Wire overhead | Larger (type info per field) | Smaller (types known a priori) |
| Reader needs schema? | No | Yes |
| Debugging | Easy (can dump any message) | Need schema to decode |
| Performance | Slightly slower (type checks) | Faster (no type dispatch) |
| Use case | Logging, debugging, ad-hoc | RPC, databases, high-throughput |

### Q2: Implement a schema fingerprint from field definitions

```cpp

#include <cstdint>
#include <iostream>
#include <vector>

uint64_t hash_combine(uint64_t seed, uint64_t value) {
    seed ^= value + 0x9e3779b97f4a7c15 + (seed << 6) + (seed >> 2);
    return seed;
}

struct FieldSchema {
    uint16_t id;
    WireType type;
    bool required;
};

uint64_t compute_fingerprint(const std::vector<FieldSchema>& schema) {
    uint64_t fp = 0;
    for (auto& f : schema) {
        fp = hash_combine(fp, f.id);
        fp = hash_combine(fp, static_cast<uint64_t>(f.type));
    }
    return fp;
}

int main() {
    std::vector<FieldSchema> v1 = {{1, WireType::String, true}, {2, WireType::Uint32, true}};
    std::vector<FieldSchema> v2 = {{1, WireType::String, true}, {2, WireType::Uint32, true},
                                    {3, WireType::Float64, false}};

    std::cout << "v1: 0x" << std::hex << compute_fingerprint(v1) << "\n";
    std::cout << "v2: 0x" << std::hex << compute_fingerprint(v2) << "\n";
}

```

### Q3: When should you prefer self-describing formats in production

Self-describing formats are preferred when:

- **Log aggregation**: heterogeneous log types from many services
- **Schema registry unavailable**: ad-hoc communication between independently deployed components
- **Debugging tools**: message inspectors that must decode without preloaded schemas
- **Migration periods**: transitioning between schema versions where both old/new messages coexist
- **Event sourcing**: stored events must be readable decades later without the original code

---

## Notes

- CBOR and MessagePack are standardized self-describing binary formats — prefer them over custom TLV.
- For high-throughput paths, schema-based formats (FlatBuffers, Protobuf) are better due to lower overhead.
- Hybrid approach: embed a schema fingerprint in the header, carry type tags only for unknown extensions.
