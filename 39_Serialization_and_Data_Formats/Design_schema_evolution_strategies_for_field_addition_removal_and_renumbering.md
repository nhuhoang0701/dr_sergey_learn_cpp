# Design schema evolution strategies for field addition, removal, and renumbering

**Category:** Serialization & Data Formats  
**Standard:** C++17  
**Reference:** <https://protobuf.dev/programming-guides/proto3/#updating>  

---

## Topic Overview

As software evolves, data schemas change. Schema evolution must allow new code to read old data and (ideally) old code to read new data.

### Evolution Operations and Their Safety

| Operation | Backward Compatible? | Forward Compatible? | Safe? |
| --- | --- | --- | --- |
| Add optional field | Yes | Yes (old readers skip) | Safe |
| Add required field | **No** (old data lacks it) | Yes | Dangerous |
| Remove optional field | Yes (new readers use default) | Yes | Safe |
| Remove required field | Yes | **No** (old readers expect it) | Dangerous |
| Rename field | **Depends** on wire format | — | Use field IDs |
| Change field type | **No** (incompatible) | **No** | Never do this |
| Reuse field number | **No** (data corruption) | **No** | Never do this |

### Strategy 1: Always-Optional Fields with Defaults

```cpp

#include <optional>
#include <string>
#include <iostream>

// Version 1
struct UserV1 {
    uint32_t id;
    std::string name;
    // field_id=1: id, field_id=2: name
};

// Version 2: added email (optional, field_id=3)
struct UserV2 {
    uint32_t id;
    std::string name;
    std::optional<std::string> email;  // New in v2 — absent in old data
};

// Version 3: deprecated age, added phone (field_id=4)
struct UserV3 {
    uint32_t id;
    std::string name;
    std::optional<std::string> email;
    // field_id=3 is RETIRED — never reused
    std::optional<std::string> phone;  // field_id=4
};

void process_user(const UserV3& user) {
    std::cout << "ID: " << user.id << "\n";
    std::cout << "Name: " << user.name << "\n";
    std::cout << "Email: " << user.email.value_or("(not provided)") << "\n";
    std::cout << "Phone: " << user.phone.value_or("(not provided)") << "\n";
}

```

### Strategy 2: Field ID Registry

```cpp

#include <cstdint>
#include <set>
#include <iostream>

// Maintain a registry of all field IDs ever used
// Mark deprecated fields as RESERVED — never reassign their IDs

namespace schema {
    // Active fields
    constexpr uint16_t FIELD_ID       = 1;
    constexpr uint16_t FIELD_NAME     = 2;
    constexpr uint16_t FIELD_EMAIL    = 3;
    constexpr uint16_t FIELD_PHONE    = 5;

    // RESERVED — formerly used, never reuse!
    constexpr uint16_t RESERVED_AGE   = 4;  // Removed in v3

    const std::set<uint16_t> reserved_ids = {4};

    bool is_reserved(uint16_t id) {
        return reserved_ids.count(id) > 0;
    }
}

```

### Strategy 3: Union-Based Schema Versioning

```cpp

#include <variant>
#include <string>

struct SchemaV1 { uint32_t id; std::string name; };
struct SchemaV2 { uint32_t id; std::string name; std::string email; };
struct SchemaV3 { uint32_t id; std::string name; std::string email; std::string phone; };

using AnySchema = std::variant<SchemaV1, SchemaV2, SchemaV3>;

// Convert old schemas to latest
SchemaV3 upgrade(const AnySchema& data) {
    return std::visit([](const auto& v) -> SchemaV3 {
        SchemaV3 result;
        result.id = v.id;
        result.name = v.name;
        if constexpr (requires { v.email; })
            result.email = v.email;
        if constexpr (requires { v.phone; })
            result.phone = v.phone;
        return result;
    }, data);
}

```

---

## Self-Assessment

### Q1: Why is changing a field's type more dangerous than removing it

Changing a field's type (e.g., `int32` to `string`) means old readers interpret new data with the wrong type — silent data corruption with no error. Removing a field with a tagged format simply means old readers see a missing field and use a default. Type changes are never backward or forward compatible.

### Q2: Design a protocol buffer-style schema with reserved fields

```protobuf

// Protobuf .proto file demonstrating reserved fields
message User {
    uint32 id = 1;
    string name = 2;
    // Field 3 was "age" — removed in v3, RESERVED forever
    reserved 3;
    reserved "age";
    string email = 4;
    string phone = 5;
}

```

### Q3: Implement a migration function between schema versions

```cpp

#include <string>
#include <optional>
#include <iostream>
#include <functional>
#include <vector>

struct UserV1 { uint32_t id; std::string name; };
struct UserV2 { uint32_t id; std::string name; std::optional<std::string> email; };

UserV2 migrate_v1_to_v2(const UserV1& v1) {
    return {v1.id, v1.name, std::nullopt}; // email defaults to absent
}

// Chain migrations for multi-version jumps
template<typename From, typename To>
using Migration = std::function<To(const From&)>;

int main() {
    UserV1 old_user{1, "Alice"};
    UserV2 new_user = migrate_v1_to_v2(old_user);
    std::cout << "Migrated: " << new_user.name
              << " email=" << new_user.email.value_or("(none)") << "\n";
}

```

---

## Notes

- **Golden rule**: never reuse field IDs and never change field types.
- Protobuf, FlatBuffers, and Avro all implement these evolution principles differently.
- `std::optional` is the natural C++ representation for "field may be absent."
- Keep a schema changelog document tracking every field addition/removal with version numbers.
