# Implement zero-copy deserialization with FlatBuffers and Cap'n Proto

**Category:** Serialization & Data Formats  
**Standard:** C++17  
**Reference:** <https://flatbuffers.dev/> · <https://capnproto.org/>  

---

## Topic Overview

**Zero-copy deserialization** means reading structured data directly from a byte buffer without parsing, copying, or allocating memory. The serialized format IS the in-memory format. This gives O(1) access time to any field.

### Traditional vs Zero-Copy

| Aspect | Traditional (JSON/Protobuf) | Zero-Copy (FlatBuffers/Cap'n Proto) |
| --- | --- | --- |
| Deserialization | Parse entire message → allocate objects | Cast pointer → done |
| Access time | O(n) parse first | O(1) immediate |
| Memory allocation | Yes (new objects) | No (read from buffer) |
| Random access | After parsing only | Immediate |
| CPU cost | High (parsing) | Near zero |

### FlatBuffers Example

FlatBuffers schema (`monster.fbs`):

```cpp

namespace Game;
table Monster {
  name: string;
  hp: int = 100;
  pos: Vec3;
  inventory: [ubyte];
}
struct Vec3 {
  x: float;
  y: float;
  z: float;
}
root_type Monster;

```

C++ usage after running `flatc --cpp monster.fbs`:

```cpp

#include "monster_generated.h"  // Generated header
#include <flatbuffers/flatbuffers.h>
#include <iostream>

// Serializing
std::vector<uint8_t> create_monster() {
    flatbuffers::FlatBufferBuilder builder(256);

    auto name = builder.CreateString("Orc");
    std::vector<uint8_t> inv_data = {1, 2, 3, 4, 5};
    auto inventory = builder.CreateVector(inv_data);

    auto monster = Game::CreateMonster(
        builder,
        name,
        150,     // hp
        &Game::Vec3(1.0f, 2.0f, 3.0f),
        inventory
    );
    builder.Finish(monster);

    // The buffer is ready — no serialization step needed
    return {builder.GetBufferPointer(),
            builder.GetBufferPointer() + builder.GetSize()};
}

// Zero-copy reading — no deserialization!
void read_monster(const uint8_t* buf, size_t len) {
    // Verify buffer integrity (optional but recommended)
    flatbuffers::Verifier verifier(buf, len);
    if (!Game::VerifyMonsterBuffer(verifier)) {
        std::cerr << "Buffer verification failed!\n";
        return;
    }

    // "Deserialize" = just cast a pointer. Zero allocation.
    auto monster = Game::GetMonster(buf);

    std::cout << "Name: " << monster->name()->c_str() << "\n";
    std::cout << "HP: " << monster->hp() << "\n";
    std::cout << "Pos: ("
              << monster->pos()->x() << ", "
              << monster->pos()->y() << ", "
              << monster->pos()->z() << ")\n";

    // Random access into inventory — no parsing
    auto inv = monster->inventory();
    std::cout << "Inventory[2]: " << static_cast<int>(inv->Get(2)) << "\n";
}

int main() {
    auto data = create_monster();
    read_monster(data.data(), data.size());
}

```

### Cap'n Proto Comparison

```cpp

// Cap'n Proto uses memory-mapped segments for truly zero-copy I/O.
// Schema (monster.capnp):
//   struct Monster {
//     name @0 :Text;
//     hp @1 :Int32 = 100;
//     pos @2 :Vec3;
//   }
//
// Key difference: Cap'n Proto supports:
// - mmap() directly on file → read fields without loading entire file
// - Promise pipelining for RPC
// - Canonical encoding for comparison
//
// FlatBuffers advantages:
// - Simpler API, smaller runtime
// - Better language support (especially mobile)
// - Optional object API for convenience

```

---

## Self-Assessment

### Q1: Explain why FlatBuffers does not need a deserialization step

FlatBuffers stores data in a binary format that matches the in-memory representation. The serialized buffer contains:

1. A **vtable** — an array of offsets to each field.
2. **Inline data** for scalars (stored at fixed offsets from the vtable).
3. **Offset-based pointers** to variable-length data (strings, vectors, nested tables).

When you call `GetMonster(buf)`, it simply interprets the buffer pointer as a `Monster*`. No copying, no allocation. Field access follows offsets: `monster->hp()` reads an `int32_t` at a known offset from the vtable.

```cpp

// Conceptually, FlatBuffers access looks like:
int32_t Monster::hp() const {
    // Read the vtable offset for field #1
    auto vtable = /* pointer to vtable */;
    auto offset = vtable[1];
    if (offset == 0) return 100;  // Default value
    // Read directly from the buffer at that offset
    return *reinterpret_cast<const int32_t*>(data_ + offset);
}

```

### Q2: Show the performance difference between JSON parsing and FlatBuffers access

```cpp

#include <chrono>
#include <iostream>
#include <string>
#include <vector>

// Simulated comparison (conceptual)
void benchmark_comparison() {
    // JSON approach:
    //   1. Receive 1KB JSON string
    //   2. Parse entire string (tokenize, validate, build DOM)
    //   3. Navigate DOM to find "hp" field
    //   4. Convert string "150" to int
    //   Total: ~1,000-10,000 ns per message

    // FlatBuffers approach:
    //   1. Receive 256-byte buffer
    //   2. auto monster = GetMonster(buf);  // pointer cast: ~1 ns
    //   3. int hp = monster->hp();          // offset read: ~2 ns
    //   Total: ~3 ns per field access

    std::cout << "JSON parse + access:        ~5,000 ns\n";
    std::cout << "FlatBuffers access:         ~3 ns\n";
    std::cout << "Speedup:                    ~1,600x\n\n";

    std::cout << "JSON memory allocation:     yes (DOM tree)\n";
    std::cout << "FlatBuffers allocation:     none (reads buffer in-place)\n";
}

int main() {
    benchmark_comparison();
}

```

### Q3: Describe when zero-copy is NOT the right choice

```cpp

// Zero-copy is NOT ideal when:
//
// 1. Data needs transformation after reading
//    - If you'll copy all fields into application objects anyway,
//      the zero-copy benefit is lost.
//
// 2. Small messages with few fields
//    - JSON/Protobuf parse time is negligible for tiny messages.
//    - FlatBuffers adds vtable overhead.
//
// 3. Human readability is important
//    - Debugging, logging, config files → JSON/YAML is better.
//
// 4. Schema evolution is complex
//    - Protobuf has more mature schema evolution tooling.
//    - FlatBuffers schema evolution is good but less flexible.
//
// 5. Cross-language support is critical
//    - JSON/Protobuf have broader language ecosystem.
//
// 6. Buffer must be validated
//    - Untrusted FlatBuffers input MUST be verified (VerifyBuffer).
//    - Verification walks the entire buffer → loses some zero-copy benefit.
//
// Decision matrix:
// - High-frequency game state updates → FlatBuffers
// - REST API responses → JSON
// - Microservice RPC → Protobuf or Cap'n Proto
// - Network packet headers → Custom TLV or FlatBuffers
// - Config files → JSON/YAML/TOML

```

---

## Notes

- **Security**: Always call `Verifier` on untrusted FlatBuffers data to prevent buffer overflow attacks.
- FlatBuffers and Cap'n Proto both support **schema evolution** (adding fields with defaults).
- For RPC, Cap'n Proto includes a built-in RPC framework; FlatBuffers is data-only (pair with gRPC).
- **mmap + Cap'n Proto**: you can memory-map a file and read fields without loading it into RAM.
- Consider alignment: FlatBuffers naturally aligns data, Cap'n Proto uses 8-byte aligned word segments.
