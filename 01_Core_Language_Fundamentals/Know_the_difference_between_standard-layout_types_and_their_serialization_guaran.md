# Know the difference between standard-layout types and their serialization guarantees

**Category:** Core Language Fundamentals  
**Reference:** <https://en.cppreference.com/w/cpp/named_req/StandardLayoutType>  

---

## Topic Overview

### What Is a Standard-Layout Type

A **standard-layout type** has a predictable memory layout that is compatible with C structs. This matters for:

- Serialization (writing/reading raw bytes)
- Interoperability with C code and hardware
- Using `offsetof()` safely
- Passing data across DLL/ABI boundaries

### Conditions For Standard-Layout

A class/struct is standard-layout if **all** of these hold:

1. **No virtual functions** and no virtual base classes.
2. **All non-static data members** have the same access control (all `public`, or all `private`, etc.).
3. **No non-static data members of reference type.**
4. **All non-static data members and base classes** are themselves standard-layout types.
5. **No two base class subobjects** of the same type (no diamond without virtual).
6. **All non-static data members are declared in the same class** in the hierarchy (either only in the most-derived class or only in one base).
7. Has no base classes with non-static data members, OR has no non-static data members itself (data in only one level).

### Standard-Layout vs. Trivially Copyable vs. POD

| Property | Meaning | Trait |
| --- | --- | --- |
| **Trivially copyable** | Can be `memcpy`'d safely | `std::is_trivially_copyable_v` |
| **Standard-layout** | Predictable C-compatible layout | `std::is_standard_layout_v` |
| **Trivial** | Default ctor/dtor/copy/move are all trivial | `std::is_trivial_v` |
| **POD** (deprecated C++20) | Both trivial AND standard-layout | `std::is_pod_v` (deprecated) |

### Why Standard-Layout Matters

1. **`offsetof()` is well-defined** — guaranteed to give the byte offset of any member.
2. **Pointer to first member == pointer to object** — `reinterpret_cast<int*>(&obj)` points to the first member if it's an `int`.
3. **C interoperability** — the layout matches what a C compiler would produce.

---

## Self-Assessment

### Q1: List the conditions and verify with `std::is_standard_layout_v`

```cpp

#include <type_traits>
#include <iostream>
#include <string>
#include <cstddef>

// Standard-layout: YES
struct Point {
    double x, y, z;
    // All public, no virtual, no references, trivial members
};
static_assert(std::is_standard_layout_v<Point>);

// Standard-layout: YES (even with methods!)
struct Rect {
    double x, y, w, h;
    double area() const { return w * h; }  // Non-virtual functions are fine
};
static_assert(std::is_standard_layout_v<Rect>);

// Standard-layout: NO — mixed access control
struct MixedAccess {
    int pub_member;
private:
    int priv_member;  // Different access → not standard-layout
};
static_assert(!std::is_standard_layout_v<MixedAccess>);

// Standard-layout: NO — virtual function
struct WithVirtual {
    int data;
    virtual void foo() {}
};
static_assert(!std::is_standard_layout_v<WithVirtual>);

// Standard-layout: NO — reference member
struct WithRef {
    int& ref;
};
static_assert(!std::is_standard_layout_v<WithRef>);

// Standard-layout: NO — non-standard-layout member
struct HasString {
    std::string name;  // std::string is NOT standard-layout
};
static_assert(!std::is_standard_layout_v<HasString>);

// Standard-layout: NO — data in both base and derived
struct Base { int x; };
struct Derived : Base { int y; };  // Data in BOTH levels
static_assert(!std::is_standard_layout_v<Derived>);

// Standard-layout: YES — data only in derived
struct EmptyBase {};
struct DerivedOk : EmptyBase { int x, y; };  // Data only in DerivedOk
static_assert(std::is_standard_layout_v<DerivedOk>);

int main() {
    std::cout << std::boolalpha;
    std::cout << "Point:       " << std::is_standard_layout_v<Point> << "\n";       // true
    std::cout << "Rect:        " << std::is_standard_layout_v<Rect> << "\n";        // true
    std::cout << "MixedAccess: " << std::is_standard_layout_v<MixedAccess> << "\n"; // false
    std::cout << "WithVirtual: " << std::is_standard_layout_v<WithVirtual> << "\n"; // false
    std::cout << "WithRef:     " << std::is_standard_layout_v<WithRef> << "\n";     // false
    std::cout << "HasString:   " << std::is_standard_layout_v<HasString> << "\n";   // false
    std::cout << "Derived:     " << std::is_standard_layout_v<Derived> << "\n";     // false
    std::cout << "DerivedOk:   " << std::is_standard_layout_v<DerivedOk> << "\n";   // true
    return 0;
}

```

### Q2: Explain why standard-layout guarantees `offsetof()` is well-defined

**`offsetof(Type, member)` returns the byte offset of `member` within `Type`.** It is only well-defined for standard-layout types because:

1. The compiler cannot insert hidden members (vtable pointers) at the beginning.
2. Member order in memory matches declaration order (no reordering across access sections).
3. The first member is guaranteed to be at offset 0 (pointer-interconvertible).

```cpp

#include <cstddef>
#include <iostream>

struct Packet {  // Standard-layout
    uint32_t header;
    uint16_t length;
    uint8_t  flags;
    uint8_t  checksum;
};
static_assert(std::is_standard_layout_v<Packet>);

int main() {
    std::cout << "offsetof(header):   " << offsetof(Packet, header)   << "\n"; // 0
    std::cout << "offsetof(length):   " << offsetof(Packet, length)   << "\n"; // 4
    std::cout << "offsetof(flags):    " << offsetof(Packet, flags)    << "\n"; // 6
    std::cout << "offsetof(checksum): " << offsetof(Packet, checksum) << "\n"; // 7
    std::cout << "sizeof(Packet):     " << sizeof(Packet)             << "\n"; // 8

    // Pointer to first member == pointer to object:
    Packet pkt{0x12345678, 100, 0xFF, 0xAB};
    uint32_t* first = reinterpret_cast<uint32_t*>(&pkt);
    std::cout << "First member via cast: 0x" << std::hex << *first << "\n"; // 0x12345678

    return 0;
}

```

**For non-standard-layout types, `offsetof()` is undefined behavior** — the compiler might add a vtable pointer before the first member, reorder members, or use different padding.

### Q3: Show a struct that breaks standard-layout by adding a virtual function

```cpp

#include <type_traits>
#include <cstddef>
#include <iostream>

// Standard-layout version:
struct SensorDataSL {
    float temperature;
    float humidity;
    uint32_t timestamp;
};
static_assert(std::is_standard_layout_v<SensorDataSL>);

// Non-standard-layout version (virtual function added):
struct SensorDataVirtual {
    float temperature;
    float humidity;
    uint32_t timestamp;

    virtual void process() {}  // This BREAKS standard-layout!
};
static_assert(!std::is_standard_layout_v<SensorDataVirtual>);

int main() {
    std::cout << "sizeof(SL):      " << sizeof(SensorDataSL) << "\n";      // 12
    std::cout << "sizeof(Virtual):  " << sizeof(SensorDataVirtual) << "\n"; // 24 (8 bytes vtable ptr + padding)

    // The virtual function causes:
    // 1. A vtable pointer (typically 8 bytes on 64-bit) is inserted BEFORE the first member
    // 2. sizeof increases by at least pointer size + potential padding
    // 3. offsetof() becomes undefined behavior
    // 4. You can't memcpy/serialize the object safely
    // 5. C interoperability is broken

    // Safe serialization of standard-layout type:
    SensorDataSL data{23.5f, 65.0f, 1000};
    unsigned char buffer[sizeof(SensorDataSL)];
    std::memcpy(buffer, &data, sizeof(data));  // Safe — standard-layout

    SensorDataSL restored;
    std::memcpy(&restored, buffer, sizeof(restored));  // Safe — trivially copyable too
    std::cout << "Restored temp: " << restored.temperature << "\n";  // 23.5

    return 0;
}

```

**Consequences of breaking standard-layout:**

- `sizeof` increases (vtable pointer inserted)
- Members shift — their offsets are no longer predictable
- `offsetof()` is UB
- `memcpy` serialization becomes unsafe (copies inner vtable pointer)
- C interoperability breaks
- Can't cast pointer-to-first-member to pointer-to-object

---

## Additional Examples

### Common Pitfall: std::string Breaks Standard-Layout

```cpp

#include <type_traits>

struct Config {
    int version;
    std::string name;     // std::string is NOT standard-layout!
    double threshold;
};
static_assert(!std::is_standard_layout_v<Config>);

// Fix: use char arrays for C-compatible structs
struct ConfigC {
    int version;
    char name[64];        // Fixed-size char array IS standard-layout
    double threshold;
};
static_assert(std::is_standard_layout_v<ConfigC>);

```

---

## Notes

- Standard-layout types can be safely passed to/from C code, hardware, and serialized via `memcpy`.
- Adding virtual functions, reference members, or mixed access control breaks standard-layout.
- Use `std::is_standard_layout_v<T>` to verify at compile time.
- `offsetof()` is only well-defined for standard-layout types.
- POD = trivial + standard-layout (deprecated in C++20, but the concept remains useful).

**How this works:**

- Show a struct that breaks standard-layout by adding a virtual function.
- Explain the consequence.

---

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
