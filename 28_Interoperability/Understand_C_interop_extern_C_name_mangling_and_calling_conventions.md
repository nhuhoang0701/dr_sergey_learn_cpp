# Understand C interop: extern C, name mangling, and calling conventions

**Category:** Interoperability  
**Item:** #590  
**Reference:** <https://en.cppreference.com/w/cpp/language/language_linkage>  

---

## Topic Overview

C++ compilers **mangle** function names to encode parameter types (enabling overloading). C compilers do not. `extern "C"` tells the C++ compiler to suppress mangling, producing a C-compatible symbol. This is the foundation of all C/C++ interop.

### Name Mangling — What Happens

```cpp

C++ source:                      Mangled symbol (GCC):
──────────                       ─────────────────────
int add(int, int)                _Z3addii
double add(double, double)       _Z3adddd
void MyClass::run(int)           _ZN7MyClass3runEi

extern "C" int add(int, int)     add        ← C-compatible

```

### Key Concepts

| Feature | C++ (default) | `extern "C"` |
| --- | :---: | :---: |
| Name mangling | Yes | **No** |
| Function overloading | Yes | **No** |
| Templates | Yes | **No** |
| Default parameters | Yes | Yes (C++ side only) |
| `noexcept` | Yes | **Required** (safety) |
| Namespace | Yes | Ignored for linkage |

### Calling Convention Defaults

| Platform | C default | C++ default | Notes |
| --- | --- | --- | --- |
| Linux x86-64 | System V AMD64 | System V AMD64 | Same — interop is easy |
| Windows x64 | `__cdecl` | `__cdecl` | Same since MSVC x64 |
| Windows x86 | `__cdecl` | `__cdecl` | But COM uses `__stdcall` |

---

## Self-Assessment

### Q1: Wrap a C++ function with extern "C" and verify the unmangled symbol with nm

**Answer:**

```cpp

// mathlib.h — C-compatible header
#ifndef MATHLIB_H
#define MATHLIB_H

#ifdef __cplusplus
extern "C" {
#endif

// These functions will have unmangled names
int add(int a, int b);
double multiply(double a, double b);

#ifdef __cplusplus
}  // extern "C"
#endif

#endif // MATHLIB_H

```

```cpp

// mathlib.cpp — C++ implementation
#include "mathlib.h"
#include <cmath>

// extern "C" suppresses mangling for these definitions
extern "C" int add(int a, int b) {
    return a + b;
}

extern "C" double multiply(double a, double b) {
    return a * b;
}

```

**Verifying with nm:**

```bash

# Compile to object file
g++ -c mathlib.cpp -o mathlib.o

# Without extern "C" — mangled:
nm mathlib.o
# T _Z3addii              ← mangled (encodes int, int)
# T _Z8multiplydd         ← mangled (encodes double, double)

# With extern "C" — unmangled:
nm mathlib.o
# T add                   ← clean C symbol
# T multiply              ← clean C symbol

# Demangle to verify:
nm mathlib.o | c++filt
# Same output with extern "C" since names are already clean

```

**Why this matters:** A C program can now link against `mathlib.o` because it looks for the symbol `add`, not `_Z3addii`.

### Q2: Explain why extern "C" functions cannot be overloaded and cannot throw C++ exceptions safely

**Answer:**

**No overloading:** Name mangling IS the mechanism that enables overloading. Without it, all overloads produce the same symbol → linker error.

```cpp

extern "C" int process(int x);       // symbol: "process"
extern "C" int process(double x);    // symbol: "process" — COLLISION!
// error: conflicting declaration of C function 'int process(double)'

```

**No safe exceptions:** C calling conventions have no stack unwinding support. If a C++ exception propagates through C code, the C stack frames are not unwound → resource leaks, undefined behavior.

```cpp

// ═══════════ WRONG: exception escapes into C ═══════════
extern "C" int dangerous(int x) {
    std::vector<int> v(x);  // might throw std::bad_alloc
    return v.size();         // if throw, C caller has no handler → UB
}

// ═══════════ CORRECT: catch all exceptions at the boundary ═══════════
extern "C" int safe(int x) noexcept {
    try {
        std::vector<int> v(x);
        return static_cast<int>(v.size());
    } catch (const std::exception& e) {
        // Log or set error code — NEVER let exception escape
        return -1;
    } catch (...) {
        return -1;
    }
}

// ═══════════ Error reporting pattern ═══════════
struct ErrorInfo {
    int code;
    char message[256];
};

extern "C" int robust_api(int input, ErrorInfo* err) noexcept {
    try {
        if (input < 0) throw std::invalid_argument("negative input");
        return input * 2;
    } catch (const std::exception& e) {
        if (err) {
            err->code = -1;
            strncpy(err->message, e.what(), sizeof(err->message) - 1);
            err->message[sizeof(err->message) - 1] = '\0';
        }
        return -1;
    } catch (...) {
        if (err) {
            err->code = -2;
            strncpy(err->message, "unknown error", sizeof(err->message) - 1);
        }
        return -1;
    }
}

```

### Q3: Write a C-compatible header using #ifdef __cplusplus to expose a C++ class as a C API

**Answer:**

```cpp

// ═══════════ image_processor.h — C-compatible header ═══════════
#ifndef IMAGE_PROCESSOR_H
#define IMAGE_PROCESSOR_H

#include <stddef.h>  // size_t (C header, not <cstddef>)

#ifdef __cplusplus
extern "C" {
#endif

// Opaque handle — C sees a pointer to incomplete type
typedef struct ImageProcessor ImageProcessor;

// Lifecycle
ImageProcessor* image_processor_create(int width, int height);
void image_processor_destroy(ImageProcessor* ip);

// Operations
int image_processor_load(ImageProcessor* ip, const char* path);
int image_processor_blur(ImageProcessor* ip, float radius);
int image_processor_save(ImageProcessor* ip, const char* path);

// Query
int image_processor_width(const ImageProcessor* ip);
int image_processor_height(const ImageProcessor* ip);
const char* image_processor_last_error(const ImageProcessor* ip);

#ifdef __cplusplus
}  // extern "C"
#endif

#endif // IMAGE_PROCESSOR_H

```

```cpp

// ═══════════ image_processor.cpp — C++ implementation ═══════════
#include "image_processor.h"
#include <string>
#include <vector>
#include <cstring>

// The actual C++ class — never exposed to C
struct ImageProcessor {
    int width_;
    int height_;
    std::vector<uint8_t> pixels_;
    std::string last_error_;

    ImageProcessor(int w, int h)
        : width_(w), height_(h), pixels_(w * h * 4, 0) {}

    bool load(const char* path) {
        // ... real implementation ...
        return true;
    }

    bool blur(float radius) {
        if (radius < 0) {
            last_error_ = "negative radius";
            return false;
        }
        // ... real implementation ...
        return true;
    }

    bool save(const char* path) {
        // ... real implementation ...
        return true;
    }
};

// ═══════════ C API implementation — catch ALL exceptions ═══════════
extern "C" {

ImageProcessor* image_processor_create(int width, int height) {
    try {
        return new ImageProcessor(width, height);
    } catch (...) {
        return nullptr;
    }
}

void image_processor_destroy(ImageProcessor* ip) {
    delete ip;  // safe if ip is nullptr
}

int image_processor_load(ImageProcessor* ip, const char* path) {
    try {
        return ip->load(path) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_blur(ImageProcessor* ip, float radius) {
    try {
        return ip->blur(radius) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_save(ImageProcessor* ip, const char* path) {
    try {
        return ip->save(path) ? 0 : -1;
    } catch (const std::exception& e) {
        ip->last_error_ = e.what();
        return -1;
    }
}

int image_processor_width(const ImageProcessor* ip) { return ip->width_; }
int image_processor_height(const ImageProcessor* ip) { return ip->height_; }
const char* image_processor_last_error(const ImageProcessor* ip) {
    return ip->last_error_.c_str();
}

}  // extern "C"

```

```c

// ═══════════ main.c — Pure C consumer ═══════════
#include "image_processor.h"
#include <stdio.h>

int main(void) {
    ImageProcessor* ip = image_processor_create(1920, 1080);
    if (!ip) {
        fprintf(stderr, "Failed to create processor\n");
        return 1;
    }

    if (image_processor_load(ip, "photo.png") != 0) {
        fprintf(stderr, "Load error: %s\n", image_processor_last_error(ip));
    }

    image_processor_blur(ip, 2.5f);
    image_processor_save(ip, "blurred.png");

    printf("Size: %dx%d\n", image_processor_width(ip), image_processor_height(ip));

    image_processor_destroy(ip);
    return 0;
}

```

---

## Notes

- **Opaque handle pattern** is the gold standard for C-API design: C gets a pointer, never sees struct internals
- `extern "C"` does NOT change the calling convention — it only changes name mangling
- On Windows, COM interfaces use `__stdcall`; regular C APIs use `__cdecl` — `extern "C"` gives `__cdecl`
- Always mark `extern "C"` functions `noexcept` or wrap bodies in `try/catch`
- Use `nm -C` (demangle) to debug symbol issues; on Windows use `dumpbin /exports`
- Prefer `#pragma once` over include guards for modern compilers, but include guards are more portable (C code)
