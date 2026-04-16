# Use `std::to_underlying` (C++23) for Safe Enum-to-Integer Conversion# Use `std::to_underlying` (C++23) for Safe Enum-to-Integer Conversion


- Compilers: GCC 11+, Clang 13+, MSVC 19.30+ (VS 2022 17.0).- Feature-test macro: `__cpp_lib_to_underlying >= 202102L`.- For bit-flag enums, `std::to_underlying` replaces the need for a separate `underlying_cast` utility that many codebases define internally.- Prefer `std::to_underlying` over `static_cast<int>`: the latter hardcodes the target type and silently truncates if the underlying type is wider than `int`.- The function is `constexpr noexcept` — zero overhead, usable in `static_assert` and template arguments.- It works with both scoped (`enum class`) and unscoped (`enum`) enumerations.- `std::to_underlying` is in `<utility>`, not `<type_traits>` — a common source of missing-include errors.## Notes---```}    static_assert(has_flag(flags, Flags::C));    static_assert(!has_flag(flags, Flags::B));    static_assert(has_flag(flags, Flags::A));        std::to_underlying(Flags::A) | std::to_underlying(Flags::C));    constexpr auto flags = static_cast<Flags>(    std::cout << r.status_code() << " — " << r.category() << '\n';    Response r{HttpStatus::NotFound};int main() {}    return (std::to_underlying(set) & std::to_underlying(flag)) != 0;constexpr bool has_flag(Flags set, Flags flag) {enum class Flags : uint8_t { A = 1, B = 2, C = 4 };// Comparison: to_underlying in bitwise operations};    }        return "Unknown";        if (code >= 500 && code < 600) return "Server Error";        if (code >= 400 && code < 500) return "Client Error";        if (code >= 200 && code < 300) return "Success";        auto code = std::to_underlying(status);        // Use to_underlying for range-based classification    std::string_view category() const noexcept {    }        return std::to_underlying(status);    uint16_t status_code() const noexcept {    HttpStatus status;struct Response {// Use to_underlying for wire-protocol encoding};    ServerError = 500,    NotFound    = 404,    OK          = 200,enum class HttpStatus : uint16_t {#endif}    }        return static_cast<std::underlying_type_t<E>>(e);    constexpr auto to_underlying(E e) noexcept {    template <typename E>namespace std {#if __cpp_lib_to_underlying < 202102L// Polyfill for pre-C++23 compilers#include <cstdint>#include <utility>#include <iostream>#include <type_traits>```cpp### Q3: How do you implement a pre-C++23 polyfill and use `to_underlying` in a switch-case exhaustiveness pattern?---**Key insight:** `std::to_underlying(LogLevel::COUNT)` in the array size guarantees the table stays in sync with the enum even if the underlying type changes.```}    }        std::cout << to_string(lvl) << '\n';         lvl = next(lvl)) {         lvl != LogLevel::COUNT;    for (auto lvl = LogLevel::Trace;    static_assert(to_string(LogLevel::Error) == "ERROR");int main() {}    return static_cast<LogLevel>(val + 1);    auto val = std::to_underlying(lvl);constexpr LogLevel next(LogLevel lvl) {// Type-safe enum increment}    return "UNKNOWN";    if (idx < level_names.size()) return level_names[idx];    auto idx = std::to_underlying(lvl);constexpr std::string_view to_string(LogLevel lvl) {};    "TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"    std::to_underlying(LogLevel::COUNT)> level_names = {constexpr std::array<std::string_view,// Lookup table indexed by enum};    Trace, Debug, Info, Warn, Error, Fatal, COUNTenum class LogLevel : uint8_t {#include <iostream>#include <string_view>#include <utility>#include <array>```cpp### Q2: How do you use `std::to_underlying` as an array index for enum-keyed lookup tables?---**Key insight:** When the underlying type changes (e.g., from `uint32_t` to `uint64_t`), every `static_cast<uint32_t>` site silently truncates. `std::to_underlying` adapts automatically.```}    serialize(Permission::All); // 0x7    serialize(perms);         // 0x3    static_assert(raw == 0x03);    constexpr auto raw = std::to_underlying(perms);    // Compile-time evaluation    constexpr auto perms = Permission::Read | Permission::Write;int main() {}              << std::hex << raw << '\n';    std::cout << "Serialized permission bits: 0x"    static_assert(std::is_same_v<decltype(raw), uint32_t>);    auto raw = std::to_underlying(p);    // CORRECT: deduces uint32_t automatically    // int raw = static_cast<int>(p);    // WRONG: static_cast<int> silently narrows if underlying is uint64_tvoid serialize(Permission p) {// Serialization helper — always uses the correct underlying type}        std::to_underlying(a) | std::to_underlying(b));    return static_cast<Permission>(constexpr Permission operator|(Permission a, Permission b) {};    All     = Read | Write | Execute  // bit-field enum    Execute = 0x04,    Write   = 0x02,    Read    = 0x01,enum class Permission : uint32_t {#include <type_traits>#include <iostream>#include <utility>#include <cstdint>```cpp### Q1: How does `std::to_underlying` compare with `static_cast` for serialization safety?## Self-Assessment---```└──────────────┘                        └──────────────────┘│  Color::Red  │                        │  (e.g. uint8_t)   ││  enum class  │ ────────────────────►  │  underlying_type  │┌──────────────┐   std::to_underlying   ┌──────────────────┐```Common use cases include serialization (writing enum values to binary/JSON), logging, interfacing with C APIs that expect integers, and indexing arrays by enum value. The function is trivially implementable — the value proposition is readability, correctness, and intent expression.| `std::to_underlying(e)` | Yes | Yes | Excellent | Yes || `static_cast<underlying_type_t<E>>(e)` | Yes | Yes | Verbose | Yes || `static_cast<int>(e)` | No — silently narrows | No — hardcodes `int` | Poor | Yes ||---|---|---|---|---|| Approach | Type-safe | Correct type deduced | Readable | Compile-time |The function is `constexpr` and `noexcept`. It participates in overload resolution only when the argument is an enumeration type, providing a compile-time guard against accidental misuse on non-enum types. This makes it strictly safer than a bare `static_cast<int>`.`std::to_underlying`, defined in `<utility>`, converts a scoped or unscoped enumeration value to its underlying integral type. Before C++23, the idiomatic way was `static_cast<std::underlying_type_t<E>>(e)` — verbose, error-prone if the target type is manually specified, and invisible to grep-based code audits. `std::to_underlying` replaces this pattern with a single, self-documenting call.## Topic Overview---**Reference:** [cppreference — std::to_underlying](https://en.cppreference.com/w/cpp/utility/to_underlying)  **Standard:** C++23  **Category:** Standard Library — New in C++23/26  

**Category:** Standard Library — New in C++23/26  
**Standard:** C++23  
**Reference:** https://en.cppreference.com/w/cpp/utility/to_underlying  

---

## Topic Overview

`std::to_underlying` is a utility function introduced in C++23 (`<utility>`) that converts a scoped or unscoped enumeration value to its underlying integer type. It replaces the verbose `static_cast<std::underlying_type_t<E>>(e)` pattern with a concise, readable, and type-safe call.

Before C++23, extracting the numeric value of an enum required either an explicit `static_cast` to the underlying type or a helper function that developers had to write themselves. Both approaches are error-prone: `static_cast` lets you cast to the *wrong* integer type silently, and hand-rolled helpers are inconsistent across codebases. `std::to_underlying` eliminates these problems by always deducing the correct underlying type.

| Approach | Type-safe? | Deduces underlying type? | Readable? |
| --- | --- | --- | --- |
| `static_cast<int>(e)` | No — allows wrong target type | No | Moderate |
| `static_cast<std::underlying_type_t<E>>(e)` | Yes | Yes | Poor — verbose |
| `std::to_underlying(e)` | Yes | Yes | Excellent |

The function is `constexpr` and `noexcept`, making it usable in compile-time contexts, template metaprogramming, serialization routines, logging, and hash functions. It works identically for scoped (`enum class`) and unscoped (`enum`) enumerations.

### Implementation (simplified)

```asm

template <class Enum>
constexpr std::underlying_type_t<Enum> to_underlying(Enum e) noexcept {
    return static_cast<std::underlying_type_t<Enum>>(e);
}

```

The simplicity is the point: the standard provides a well-named, discoverable vocabulary function so that every codebase converges on one idiom.

---

## Self-Assessment

### Q1: How does `std::to_underlying` compare to `static_cast` in a serialization context, and what bugs does it prevent

```cpp

#include <cstdint>
#include <utility>
#include <iostream>
#include <vector>
#include <bit>

enum class Permission : std::uint8_t {
    Read    = 1,
    Write   = 2,
    Execute = 4,
    Admin   = 128
};

// BUG-PRONE: static_cast to the wrong type silently widens
void serialize_bad(Permission p, std::vector<std::byte>& buf) {
    // Developer guessed int (4 bytes) instead of uint8_t (1 byte).
    // Compiles fine, writes 4 bytes instead of 1.
    auto val = static_cast<int>(p);
    auto* raw = reinterpret_cast<const std::byte*>(&val);
    buf.insert(buf.end(), raw, raw + sizeof(val));
}

// CORRECT: std::to_underlying always yields uint8_t here
void serialize_good(Permission p, std::vector<std::byte>& buf) {
    auto val = std::to_underlying(p);  // deduced as uint8_t
    static_assert(std::same_as<decltype(val), std::uint8_t>);
    buf.push_back(static_cast<std::byte>(val));
}

int main() {
    std::vector<std::byte> buf;

    serialize_good(Permission::Admin, buf);
    std::cout << "Serialized size (good): " << buf.size() << " byte(s)\n";
    // Output: 1

    buf.clear();
    serialize_bad(Permission::Admin, buf);
    std::cout << "Serialized size (bad):  " << buf.size() << " byte(s)\n";
    // Output: 4 — silent data bloat and protocol mismatch
}

```

**Key insight:** `std::to_underlying` eliminates an entire class of serialization bugs where the developer casts to a mismatched integer width.

---

### Q2: How do you use `std::to_underlying` inside a `constexpr` bitmask flag system

```cpp

#include <cstdint>
#include <utility>
#include <type_traits>
#include <iostream>
#include <format>

enum class Style : std::uint16_t {
    None      = 0,
    Bold      = 1 << 0,
    Italic    = 1 << 1,
    Underline = 1 << 2,
    Strike    = 1 << 3,
    Code      = 1 << 4,
};

// Overload bitwise operators using to_underlying for clarity
constexpr Style operator|(Style a, Style b) noexcept {
    return static_cast<Style>(
        std::to_underlying(a) | std::to_underlying(b));
}

constexpr Style operator&(Style a, Style b) noexcept {
    return static_cast<Style>(
        std::to_underlying(a) & std::to_underlying(b));
}

constexpr bool has_flag(Style set, Style flag) noexcept {
    return (set & flag) != Style::None;
}

// Compile-time validation
static_assert(std::to_underlying(Style::Bold) == 1);
static_assert(std::to_underlying(Style::Code) == 16);
static_assert(has_flag(Style::Bold | Style::Italic, Style::Italic));

// Logging / formatting helper
void print_styles(Style s) {
    auto raw = std::to_underlying(s);
    std::cout << std::format("Style mask: 0x{:04X} (decimal {})\n", raw, raw);
    if (has_flag(s, Style::Bold))      std::cout << "  + Bold\n";
    if (has_flag(s, Style::Italic))    std::cout << "  + Italic\n";
    if (has_flag(s, Style::Underline)) std::cout << "  + Underline\n";
    if (has_flag(s, Style::Strike))    std::cout << "  + Strike\n";
    if (has_flag(s, Style::Code))      std::cout << "  + Code\n";
}

int main() {
    constexpr auto heading = Style::Bold | Style::Underline;
    print_styles(heading);
    // Style mask: 0x0005 (decimal 5)
    //   + Bold
    //   + Underline
}

```

---

### Q3: How do you use `std::to_underlying` as a hash or map key for enum-indexed data structures

```cpp

#include <utility>
#include <unordered_map>
#include <string>
#include <iostream>
#include <format>
#include <array>
#include <cstdint>

enum class LogLevel : std::uint8_t {
    Trace = 0,
    Debug = 1,
    Info  = 2,
    Warn  = 3,
    Error = 4,
    Fatal = 5,
    COUNT // sentinel for array sizing
};

// 1. Compile-time array indexed by enum via to_underlying
constexpr std::array<const char*, std::to_underlying(LogLevel::COUNT)>
    level_names = {"TRACE", "DEBUG", "INFO", "WARN", "ERROR", "FATAL"};

constexpr const char* to_string(LogLevel lv) noexcept {
    auto idx = std::to_underlying(lv);
    if (idx < level_names.size()) return level_names[idx];
    return "UNKNOWN";
}

// 2. Hash functor using to_underlying — generic for any enum
struct EnumHash {
    template <typename E>
        requires std::is_enum_v<E>
    constexpr std::size_t operator()(E e) const noexcept {
        return std::hash<std::underlying_type_t<E>>{}(std::to_underlying(e));
    }
};

int main() {
    // Compile-time lookup
    static_assert(to_string(LogLevel::Warn)[0] == 'W');

    // Runtime: enum-keyed map with generic hash
    std::unordered_map<LogLevel, std::string, EnumHash> thresholds;
    thresholds[LogLevel::Error] = "page oncall";
    thresholds[LogLevel::Fatal] = "page oncall + manager";

    for (auto& [level, action] : thresholds) {
        std::cout << std::format("[{}] -> {}\n", to_string(level), action);
    }

    // Direct use in switch (showing exhaustive pattern)
    LogLevel current = LogLevel::Info;
    switch (std::to_underlying(current)) {
        case 0: case 1: std::cout << "verbose\n"; break;
        case 2: case 3: std::cout << "standard\n"; break;
        case 4: case 5: std::cout << "critical\n"; break;
    }
}

```

---

## Notes

- **Header:** `<utility>` — no additional includes needed.
- **`constexpr` + `noexcept`:** usable in every evaluation context including `static_assert`, template arguments, and `consteval` functions.
- **Works with unscoped enums too:** although less commonly needed since unscoped enums implicitly convert to integers, `to_underlying` still guarantees the *exact* underlying type rather than a potentially widened `int`.
- **Migration tip:** search your codebase for `static_cast<.*underlying_type` — every hit is a candidate for replacement with `std::to_underlying`.
- **Compiler support (as of 2025):** GCC 13+, Clang 16+, MSVC 19.34+ (VS 2022 17.4+).
- **Pair with `std::format`:** `std::to_underlying` feeds directly into format specifiers — `std::format("{}", std::to_underlying(e))` — useful for logging and diagnostics.
