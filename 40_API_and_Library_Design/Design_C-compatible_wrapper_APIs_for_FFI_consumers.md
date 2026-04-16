# Design C-compatible wrapper APIs for FFI consumers

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://isocpp.github.io/CppCoreGuidelines/CppCoreGuidelines#cpl-c-style-programming>  

---

## Topic Overview

To use a C++ library from Python, Rust, C#, or other languages, you need a C ABI boundary. This means `extern "C"` functions with no C++ types in the signature.

### The Wrapper Pattern

```cpp

// mylib.h — C header consumed by FFI users
#ifndef MYLIB_H
#define MYLIB_H

#include <stddef.h>
#include <stdint.h>

#ifdef __cplusplus
extern "C" {
#endif

typedef struct mylib_connection mylib_connection;  // Opaque handle

typedef enum {
    MYLIB_OK = 0,
    MYLIB_ERR_INVALID_ARG = 1,
    MYLIB_ERR_CONNECT = 2,
    MYLIB_ERR_TIMEOUT = 3,
} mylib_error;

// Lifecycle
mylib_error mylib_connection_create(const char* host, uint16_t port,
                                     mylib_connection** out);
void mylib_connection_destroy(mylib_connection* conn);

// Operations
mylib_error mylib_connection_send(mylib_connection* conn,
                                   const uint8_t* data, size_t len);
mylib_error mylib_connection_recv(mylib_connection* conn,
                                   uint8_t* buf, size_t buflen, size_t* received);

// Version
const char* mylib_version(void);

#ifdef __cplusplus
}
#endif
#endif

// mylib_wrapper.cpp — C++ implementation
#include "mylib.h"
#include "connection.hpp"  // Internal C++ class

struct mylib_connection {
    mylib::Connection impl;
};

extern "C" {

mylib_error mylib_connection_create(const char* host, uint16_t port,
                                     mylib_connection** out) {
    if (!host || !out) return MYLIB_ERR_INVALID_ARG;
    try {
        auto* conn = new mylib_connection{mylib::Connection(host, port)};
        *out = conn;
        return MYLIB_OK;
    } catch (const std::exception&) {
        return MYLIB_ERR_CONNECT;
    }
}

void mylib_connection_destroy(mylib_connection* conn) {
    delete conn;
}

const char* mylib_version(void) {
    return "1.0.0";
}

}

```

---

## Self-Assessment

### Q1: Why use opaque pointers instead of exposing struct layout

Opaque pointers (`typedef struct X X;` without definition) hide the C++ internal layout from FFI consumers. This provides ABI stability — you can change the internal struct freely without breaking binary compatibility. It also prevents FFI users from accessing private members.

### Q2: Why must exceptions never cross the extern "C" boundary

The C ABI has no exception mechanism. Propagating a C++ exception through C code (or into Python/Rust) causes undefined behavior (typically a crash). Every `extern "C"` function must catch all exceptions and convert them to error codes.

### Q3: Show how Python calls this API via ctypes

```python

import ctypes

lib = ctypes.CDLL("./libmylib.so")

# Define argument/return types
lib.mylib_connection_create.argtypes = [ctypes.c_char_p, ctypes.c_uint16,
                                         ctypes.POINTER(ctypes.c_void_p)]
lib.mylib_connection_create.restype = ctypes.c_int

conn = ctypes.c_void_p()
rc = lib.mylib_connection_create(b"localhost", 8080, ctypes.byref(conn))
if rc != 0:
    print(f"Error: {rc}")
#
lib.mylib_connection_destroy(conn)

```

---

## Notes

- Follow a consistent naming convention: `libname_type_verb` (e.g., `mylib_connection_create`).
- Always provide a `_destroy` function for every `_create`.
- Return error codes, never throw across the boundary.
- Strings must be null-terminated C strings (`const char*`).
- For buffer output, use the `(buf, buflen, actual_len)` triple pattern.
