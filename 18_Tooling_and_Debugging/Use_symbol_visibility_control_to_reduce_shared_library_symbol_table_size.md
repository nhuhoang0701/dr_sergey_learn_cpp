# Use symbol visibility control to reduce shared library symbol table size

**Category:** Tooling & Debugging  
**Item:** #414  
**Reference:** <https://gcc.gnu.org/wiki/Visibility>  

---

## Topic Overview

By default, all symbols in a shared library are exported. Controlling visibility reduces binary size, improves load time, and prevents symbol collisions.

```cpp

Default (all visible):           Hidden + selective export:
  libfoo.so exports:               libfoo.so exports:
    foo_public()                     foo_public()      (exported)
    foo_internal()    <-- leaked!    
    helper_func()     <-- leaked!    (everything else hidden)
    _ZN3Bar...        <-- leaked!    
  Symbol table: 500 entries          Symbol table: 10 entries
  Load time: slow                    Load time: fast

```

---

## Self-Assessment

### Q1: Compile with `-fvisibility=hidden` and export specific symbols

```cpp

// mylib.h — public API header
#pragma once

// Mark symbols for export:
#if defined(_MSC_VER)
  #define MYLIB_API __declspec(dllexport)
#else
  #define MYLIB_API __attribute__((visibility("default")))
#endif

// Public API (exported):
MYLIB_API int compute(int x);
MYLIB_API void initialize();

class MYLIB_API Widget {  // entire class exported
public:
    void doWork();
    int getValue() const;
private:
    int value_ = 0;
};

```

```cpp

// mylib.cpp — implementation
#include "mylib.h"
#include <iostream>

// Internal helper — NOT exported (hidden by default)
namespace {
    int internal_transform(int x) {
        return x * x + 1;
    }
}

// Another internal function — also hidden
static void setup_internals() {
    std::cout << "Setup\n";
}

// Exported functions:
int compute(int x) {
    return internal_transform(x);  // calls hidden helper
}

void initialize() {
    setup_internals();
}

void Widget::doWork() { value_ = compute(value_); }
int Widget::getValue() const { return value_; }

```

```bash

# Compile with hidden visibility:
$ g++ -std=c++20 -shared -fPIC -fvisibility=hidden \
    -o libmylib.so mylib.cpp

# Check exported symbols:
$ nm -D libmylib.so | grep ' T '
# T compute
# T initialize
# T Widget::doWork()
# T Widget::getValue() const
# Only 4 exported symbols! Internal helpers are hidden

# Without -fvisibility=hidden:
$ g++ -std=c++20 -shared -fPIC -o libmylib_all.so mylib.cpp
$ nm -D libmylib_all.so | grep ' T ' | wc -l
# 15+  (internal helpers, typeinfo, vtables all exported)

```

### Q2: Benefits of hidden visibility

```cpp

Benefits:

1. SMALLER BINARY:
   - Symbol table reduced (fewer entries to resolve)
   - Fewer relocations in .rela.dyn section
   - Typical: 10-30% smaller .so file

2. FASTER LOAD TIME:
   - Dynamic linker resolves fewer symbols at dlopen()
   - Fewer GOT/PLT entries
   - Shared library loads faster

3. BETTER OPTIMIZATION:
   - Compiler knows hidden functions can't be interposed
   - Can inline hidden functions across TUs
   - Can place hidden functions in any order

4. NO SYMBOL CLASHES:
   - Two .so files can have internal functions with same name
   - Without visibility: linker picks one randomly -> silent bugs!
   - With hidden visibility: each .so uses its own copy

5. API CLARITY:
   - Only intended public API is visible
   - Prevents accidental dependency on internals

```

```bash

# Measure the difference:
$ ls -la libmylib_hidden.so    # 42,000 bytes
$ ls -la libmylib_default.so   # 56,000 bytes  (33% larger)

$ readelf -d libmylib_hidden.so | grep NEEDED
$ readelf --dyn-syms libmylib_hidden.so | wc -l    # 12
$ readelf --dyn-syms libmylib_default.so | wc -l   # 45

```

### Q3: Cross-platform visibility macro

```cpp

// export.h — works on GCC, Clang, and MSVC
#pragma once

#if defined(_MSC_VER)
  // MSVC: use __declspec
  #ifdef MYLIB_BUILDING
    #define MYLIB_EXPORT __declspec(dllexport)
  #else
    #define MYLIB_EXPORT __declspec(dllimport)
  #endif
  #define MYLIB_LOCAL
#elif defined(__GNUC__) || defined(__clang__)
  // GCC/Clang: use visibility attributes
  #define MYLIB_EXPORT __attribute__((visibility("default")))
  #define MYLIB_LOCAL  __attribute__((visibility("hidden")))
#else
  // Unknown compiler: export everything
  #define MYLIB_EXPORT
  #define MYLIB_LOCAL
#endif

```

```cpp

// Usage:
#include "export.h"

// Public API:
class MYLIB_EXPORT PublicWidget {
public:
    void compute();
};

// Explicitly hidden (even if -fvisibility=default):
class MYLIB_LOCAL InternalHelper {
public:
    void setup();
};

MYLIB_EXPORT int public_function();
MYLIB_LOCAL void internal_function();

```

```cmake

# CMake integration:
add_library(mylib SHARED mylib.cpp)
target_compile_definitions(mylib PRIVATE MYLIB_BUILDING)

# GCC/Clang: hide all symbols by default
set_target_properties(mylib PROPERTIES
    CXX_VISIBILITY_PRESET hidden
    VISIBILITY_INLINES_HIDDEN YES
)

# CMake has built-in generate_export_header:
include(GenerateExportHeader)
generate_export_header(mylib)
# Creates: mylib_export.h with MYLIB_EXPORT/MYLIB_NO_EXPORT macros

```

---

## Notes

- Always compile shared libraries with `-fvisibility=hidden`.
- Use `VISIBILITY_INLINES_HIDDEN` to hide inline function symbols.
- CMake's `GenerateExportHeader` generates portable export macros automatically.
- On Windows, symbols are hidden by default (opposite of Linux).
- Use `__attribute__((visibility("default")))` on classes to export vtable and typeinfo.
