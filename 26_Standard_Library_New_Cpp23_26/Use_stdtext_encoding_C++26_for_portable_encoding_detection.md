# Use std::text_encoding (C++26) for portable encoding detection

**Category:** Standard Library — New in C++23/26  
**Item:** #578  
**Standard:** C++26  
**Reference:** <https://en.cppreference.com/w/cpp/locale/text_encoding>  

---

## Topic Overview

This file focuses on the **compile-time vs runtime encoding distinction** and building a portable UTF-8 input handler. (See also file #762 for core `std::text_encoding` API and Windows-specific examples.)

### Two Encodings in Every Program

| Encoding                          | When decided    | API                              |
| --- | --- | --- |
| **Literal encoding**              | Compile time    | `std::text_encoding::literal()`  |
| **System/environment encoding**   | Runtime         | `std::text_encoding::environment()` |

```cpp

Source code: const char* s = "hello café";
                                    │
Compiler flag (-fexec-charset=utf-8)│
                                    ▼
Binary contains: UTF-8 bytes [68 65 6c 6c 6f 20 63 61 66 c3 a9]
                                    │
Runtime: console uses windows-1252  │
                                    ▼
Display: garbled "cafÃ©" unless converted!

```

---

## Self-Assessment

### Q1: Use std::text_encoding::environment() to detect the system's narrow character encoding at runtime

**Answer:**

```cpp

#include <text_encoding>  // C++26
#include <iostream>
#include <print>

int main() {
    // ═══════════ Detect system encoding ═══════════
    auto sys = std::text_encoding::environment();
    std::println("System encoding: {} (MIB: {})",
                 sys.name(), static_cast<int>(sys.mib()));

    // ═══════════ Common MIB checks ═══════════
    switch (sys.mib()) {
        case std::text_encoding::id::UTF8:
            std::println("System uses UTF-8 — no conversion needed");
            break;
        case std::text_encoding::id::ISO8859_1:
            std::println("System uses ISO-8859-1 (Latin-1)");
            break;
        case std::text_encoding::id::unknown:
            std::println("Unknown encoding — proceed with caution");
            break;
        default:
            std::println("System uses: {}", sys.name());
            break;
    }

    // ═══════════ Check specific encoding by name ═══════════
    auto expected = std::text_encoding("UTF-8");
    if (sys == expected) {
        std::println("Confirmed: system is UTF-8");
    }

    // ═══════════ aliases() — all known names for this encoding ═══════════
    std::println("Known aliases:");
    for (auto alias : sys.aliases()) {
        std::println("  - {}", alias);
    }
    // e.g., for UTF-8: "UTF-8", "utf-8", "csUTF8"
}

```

### Q2: Explain the difference between literal encoding (compile time) and system encoding (runtime)

**Answer:**

```cpp

#include <text_encoding>  // C++26
#include <iostream>
#include <print>

int main() {
    // ═══════════ Literal encoding: decided at COMPILE TIME ═══════════
    auto lit = std::text_encoding::literal();
    std::println("Literal encoding: {}", lit.name());
    // Determined by:
    //   MSVC:  /utf-8 or /source-charset:utf-8 /execution-charset:utf-8
    //   GCC:   -fexec-charset=utf-8 (default is UTF-8 since GCC 10)
    //   Clang: -fexec-charset=utf-8 (default is UTF-8)

    // This affects how string literals are encoded in the binary:
    const char* greeting = "café";
    // With -fexec-charset=utf-8:  bytes = 63 61 66 c3 a9 (5 bytes, UTF-8)
    // With -fexec-charset=latin1: bytes = 63 61 66 e9    (4 bytes, Latin-1)

    // ═══════════ System encoding: decided at RUNTIME ═══════════
    auto sys = std::text_encoding::environment();
    std::println("System encoding:  {}", sys.name());
    // Determined by:
    //   Linux:   LC_CTYPE locale (usually UTF-8)
    //   Windows: GetACP() → codepage (often 1252 or 65001)
    //   macOS:   UTF-8 (always)

    // ═══════════ The mismatch problem ═══════════
    if (lit != sys) {
        std::println("⚠ MISMATCH: literal={} vs system={}",
                     lit.name(), sys.name());
        std::println("  String literals will display incorrectly without conversion!");

        // Example scenario:
        // Compiled on Linux (UTF-8 literals)
        // Running on Windows (system = windows-1252)
        // → "café" stored as UTF-8 in binary
        // → Console expects windows-1252
        // → "cafÃ©" displayed (mojibake!)

        // Solution: detect at startup and configure conversion
    } else {
        std::println("✓ Match: both literal and system use {}", lit.name());
    }

    /*
    Summary:
    ┌─────────────────┬───────────────────┬──────────────────────────┐
    │                 │ Literal encoding  │ System encoding          │
    ├─────────────────┼───────────────────┼──────────────────────────┤
    │ When decided?   │ Compile time      │ Runtime                  │
    │ Changed by?     │ Compiler flags    │ OS locale / env vars     │
    │ Affects?        │ String literals   │ Console, file I/O, APIs  │
    │ Can differ?     │ YES — this causes │ encoding mismatch bugs!  │
    │ Queried via?    │ literal()         │ environment()            │
    └─────────────────┴───────────────────┴──────────────────────────┘
    */
}

```

### Q3: Show a portable UTF-8 input handler that converts from the system encoding using text_encoding

**Answer:**

```cpp

#include <text_encoding>  // C++26
#include <iostream>
#include <string>
#include <print>
#include <stdexcept>

// ═══════════ Portable UTF-8 input handler ═══════════
// Reads input from stdin, converts to UTF-8 if system encoding differs

class Utf8InputHandler {
    bool needs_conversion_;
    std::string sys_encoding_;

public:
    Utf8InputHandler() {
        auto sys = std::text_encoding::environment();
        auto target = std::text_encoding("UTF-8");
        needs_conversion_ = (sys != target);
        sys_encoding_ = sys.name();

        if (needs_conversion_) {
            std::println(std::cerr, "Note: system encoding is {} — will convert to UTF-8",
                         sys_encoding_);
        }
    }

    // Read a line and return it as UTF-8
    std::string read_line_utf8() {
        std::string line;
        if (!std::getline(std::cin, line))
            return {};

        if (!needs_conversion_)
            return line;  // Already UTF-8, no conversion needed

        return convert_to_utf8(line);
    }

private:
    std::string convert_to_utf8(const std::string& input) {
        // Platform-specific conversion
        // In production, use ICU, iconv, or OS APIs
#ifdef _WIN32
        // Windows: MultiByteToWideChar(CP_ACP) → WideCharToMultiByte(CP_UTF8)
        // Pseudocode:
        // int wide_len = MultiByteToWideChar(CP_ACP, 0, input.c_str(), -1, nullptr, 0);
        // std::wstring wide(wide_len, 0);
        // MultiByteToWideChar(CP_ACP, 0, input.c_str(), -1, wide.data(), wide_len);
        // int utf8_len = WideCharToMultiByte(CP_UTF8, 0, wide.c_str(), -1, nullptr, 0, ...);
        // std::string utf8(utf8_len, 0);
        // WideCharToMultiByte(CP_UTF8, 0, wide.c_str(), -1, utf8.data(), utf8_len, ...);
        // return utf8;
        return input;  // Simplified
#else
        // POSIX: use iconv
        // iconv_t cd = iconv_open("UTF-8", sys_encoding_.c_str());
        // ... iconv(cd, &inbuf, &inlen, &outbuf, &outlen) ...
        // iconv_close(cd);
        return input;  // On Linux/macOS, usually already UTF-8
#endif
    }
};

int main() {
    Utf8InputHandler handler;

    std::println("Enter text (will be read as UTF-8):");
    std::string line = handler.read_line_utf8();
    std::println("UTF-8 input: {}", line);
    std::println("Byte length: {}", line.size());

    // Verify UTF-8: check for valid byte sequences
    bool valid = true;
    for (size_t i = 0; i < line.size(); ++i) {
        unsigned char c = static_cast<unsigned char>(line[i]);
        if (c >= 0x80) {
            // Multi-byte UTF-8: verify continuation bytes
            int expected = (c >= 0xF0) ? 3 : (c >= 0xE0) ? 2 : (c >= 0xC0) ? 1 : -1;
            if (expected < 0) { valid = false; break; }
            for (int j = 0; j < expected && i + 1 < line.size(); ++j) {
                if ((static_cast<unsigned char>(line[++i]) & 0xC0) != 0x80) {
                    valid = false; break;
                }
            }
        }
    }
    std::println("Valid UTF-8: {}", valid ? "yes" : "no");
}

```

---

## Notes

- `std::text_encoding` detects but does not convert — conversion still requires ICU / iconv / OS APIs
- Always compile with `/utf-8` (MSVC) or `-fexec-charset=utf-8` to ensure literal encoding is UTF-8
- `text_encoding::environment()` may return different results based on `LC_CTYPE`, `LANG`, or Windows code page
- C++26 also provides `std::text_encoding::wide_literal()` for the wide string literal encoding
- On modern Linux/macOS, `environment()` almost always returns UTF-8; Windows is the main edge case
