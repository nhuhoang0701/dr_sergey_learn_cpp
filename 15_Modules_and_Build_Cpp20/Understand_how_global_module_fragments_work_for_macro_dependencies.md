# Understand how global module fragments work for macro dependencies

**Category:** Modules & Build (C++20)  
**Item:** #516  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/modules>  

---

## Topic Overview

This topic focuses on **practical macro dependency scenarios** — wrapping C APIs, conditional compilation, and platform-specific includes that require macros inside module code.

### Common Macro Dependencies That Require Global Module Fragments

| Dependency | Header | Macros needed |
| --- | --- | --- |
| POSIX API | `<unistd.h>`, `<fcntl.h>` | `O_RDONLY`, `S_IRUSR`, etc. |
| Windows API | `<windows.h>` | `NOMINMAX`, `WIN32_LEAN_AND_MEAN` |
| OpenGL | `<GL/gl.h>` | `GL_TRIANGLES`, `GL_FLOAT` |
| SQLite | `<sqlite3.h>` | `SQLITE_OK`, `SQLITE_ROW` |
| errno | `<cerrno>` | `errno`, `ENOENT`, `EACCES` |
| Configuration | `"config.h"` | `#define`-based feature flags |

### Why Macros Can't Live After `export module`

```cpp

export module wrapper;

// This is implementation-defined and likely won't work as expected:
#include <cerrno>       // ❌ macros may not be available after module declaration

export int get_errno() {
    return errno;       // errno is a macro — may fail!
}

```

**Correct approach:**

```cpp

module;                 // ← global module fragment
#include <cerrno>       // ✅ macros are guaranteed available

export module wrapper;

export int get_errno() {
    return errno;       // ✅ works — macro was included in fragment
}

```

---

## Self-Assessment

### Q1: Use `module; ... export module foo;` to include legacy headers whose macros are needed

```cpp

// file_wrapper.cppm — wraps POSIX file operations as a C++20 module
module;

// Global module fragment: include C headers with macros
#include <cstdio>      // FILE*, fopen, fclose, fread, etc.
#include <cerrno>      // errno macro
#include <cstring>     // strerror

// Platform detection (preprocessor in global fragment)
#ifdef _WIN32
#  include <io.h>
#  define PATH_SEP '\\'
#else
#  define PATH_SEP '/'
#endif

export module file_wrapper;  // ends the global module fragment

export enum class FileError {
    None,
    NotFound,
    PermissionDenied,
    Unknown
};

export struct FileResult {
    bool ok;
    FileError error;
    int error_code;  // raw errno value
};

export class FileReader {
    std::FILE* fp_ = nullptr;
public:
    FileReader() = default;
    ~FileReader() { close(); }

    // Non-copyable, movable
    FileReader(const FileReader&) = delete;
    FileReader& operator=(const FileReader&) = delete;
    FileReader(FileReader&& other) noexcept : fp_(other.fp_) { other.fp_ = nullptr; }

    FileResult open(const char* path) {
        close();
        fp_ = std::fopen(path, "r");  // C API from <cstdio>
        if (!fp_) {
            int err = errno;          // errno macro from <cerrno>
            FileError fe = FileError::Unknown;
            if (err == ENOENT)  fe = FileError::NotFound;       // macro
            if (err == EACCES)  fe = FileError::PermissionDenied; // macro
            return {false, fe, err};
        }
        return {true, FileError::None, 0};
    }

    void close() {
        if (fp_) { std::fclose(fp_); fp_ = nullptr; }
    }

    bool is_open() const { return fp_ != nullptr; }

    export char path_separator() { return PATH_SEP; } // uses platform macro
};

```

**Consumer:**

```cpp

import file_wrapper;
#include <iostream>

int main() {
    FileReader reader;
    auto result = reader.open("nonexistent.txt");

    if (!result.ok) {
        std::cout << "Error: ";
        switch (result.error) {
        case FileError::NotFound:
            std::cout << "File not found"; break;
        case FileError::PermissionDenied:
            std::cout << "Permission denied"; break;
        default:
            std::cout << "Unknown error"; break;
        }
        std::cout << " (errno=" << result.error_code << ")\n";
    }

    // ENOENT, errno, PATH_SEP are NOT visible here
    // Only the exported types and functions are accessible
}
// Expected output (typical):
// Error: File not found (errno=2)

```

**How this works:**

- C API macros (`errno`, `ENOENT`, `EACCES`) are included in the global module fragment.
- The module wraps raw C macros into type-safe C++ abstractions (`FileError` enum).
- Importers never see the raw macros — they use the clean exported interface.

### Q2: Explain why macros defined after the module declaration are not exported

Modules use a fundamentally different model from headers:

```cpp

module;
#define BEFORE_DECL 1      // Available in the module unit
export module example;

#define AFTER_DECL 2        // Also available in the module unit

export int use_macros() {
    return BEFORE_DECL + AFTER_DECL;  // Both work inside the module: returns 3
}

```

**Why macros don't export:**

1. **Macros are preprocessor concepts.** Modules operate at the **compiler level**, after preprocessing. The compiler's module system doesn't know about macros.

2. **The BMI (Binary Module Interface) stores declarations, not preprocessor state.** When `import example;` reads the BMI, it gets:
   - Function signature: `int use_macros()`
   - But NOT: `#define BEFORE_DECL 1` or `#define AFTER_DECL 2`

3. **This is intentional.** The entire point of modules is to eliminate macro leakage:

```cpp

Header model:     source text ──preprocessor──> expanded text ──compiler──> object
                       ↑
                  macros leak through #include

Module model:     module source ──> BMI (declarations only) ──> importer
                                     ↑
                                no macro state stored

```

Consumer verification:

```cpp

import example;
#include <iostream>

int main() {
    std::cout << use_macros() << '\n';  // 3 — function works

    #ifdef BEFORE_DECL
        std::cout << "BEFORE_DECL visible\n";  // NOT printed
    #endif
    #ifdef AFTER_DECL
        std::cout << "AFTER_DECL visible\n";   // NOT printed
    #endif

    std::cout << "No macros leaked\n";
}
// Expected output:
// 3
// No macros leaked

```

### Q3: Show a case where a global module fragment is required to make a C API usable from a module

**Wrapping SQLite3 C API:**

```cpp

// sqlite_wrapper.cppm
module;

// SQLite3 is a C library that uses macros extensively
#include <sqlite3.h>   // SQLITE_OK, SQLITE_ROW, SQLITE_DONE, etc.
#include <cstring>     // std::strlen

export module sqlite_wrapper;

import <string>;       // or #include <string> in global fragment
import <stdexcept>;

export class Database {
    sqlite3* db_ = nullptr;

public:
    Database() = default;
    ~Database() { if (db_) sqlite3_close(db_); }

    void open(const char* path) {
        int rc = sqlite3_open(path, &db_);
        if (rc != SQLITE_OK) {          // SQLITE_OK is a macro from sqlite3.h
            std::string msg = sqlite3_errmsg(db_);
            sqlite3_close(db_);
            db_ = nullptr;
            throw std::runtime_error("Failed to open DB: " + msg);
        }
    }

    void execute(const char* sql) {
        char* err_msg = nullptr;
        int rc = sqlite3_exec(db_, sql, nullptr, nullptr, &err_msg);
        if (rc != SQLITE_OK) {          // macro comparison
            std::string msg(err_msg);
            sqlite3_free(err_msg);
            throw std::runtime_error("SQL error: " + msg);
        }
    }

    bool is_open() const { return db_ != nullptr; }
};

// Without the global module fragment, SQLITE_OK, SQLITE_ROW etc.
// would not be available, making the C API unusable.

```

**Why it's required:**

| Without global module fragment | With global module fragment |
| --- | --- |
| `#include <sqlite3.h>` after `export module` | `#include <sqlite3.h>` in `module;` block |
| `SQLITE_OK` may not be defined | `SQLITE_OK` is guaranteed defined |
| Code doesn't compile | Code compiles correctly |
| C API unusable from module | C API fully usable |

The pattern applies to **any C library** that relies on macros: OpenSSL, zlib, POSIX, Win32, Vulkan, etc.

---

## Notes

- The global module fragment can **only** contain preprocessor directives — no actual C++ code.
- Always prefer wrapping C macros in `constexpr` variables or `enum` values in the exported interface.
- Multiple `#include` directives in the global fragment are fine — order matters as with normal `#include`.
- The global fragment is the recommended migration path for codebases heavily dependent on C libraries.
