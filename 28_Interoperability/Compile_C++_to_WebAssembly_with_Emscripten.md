# Compile C++ to WebAssembly with Emscripten

**Category:** Interoperability  
**Item:** #593  
**Standard:** C++20  
**Reference:** <https://emscripten.org/>  

---

## Topic Overview

**Emscripten** compiles C/C++ to **WebAssembly (Wasm)**, letting you run native code in the browser at near-native speed. It provides a full POSIX-like environment, libc/libc++, and bridges to JavaScript APIs.

### Compilation Pipeline

```cpp

   C++ source (.cpp)
        │
        ▼  emcc (Clang/LLVM)
   LLVM IR (.bc)
        │
        ▼  LLVM Wasm backend
   WebAssembly (.wasm) + JavaScript glue (.js)
        │
        ▼  Browser / Node.js
   Runs in Wasm VM

```

### Key Emscripten Features

| Feature | Flag / API | Purpose |
| --- | --- | --- |
| `cwrap`/`ccall` | `-sEXPORTED_FUNCTIONS` | Call C-style functions from JS |
| Embind | `--bind` | Expose C++ classes to JS |
| Asyncify | `-sASYNCIFY` | Async JS calls from sync C++ |
| File system | `-sFORCE_FILESYSTEM` | Virtual FS (MEMFS, IDBFS) |
| Threads | `-pthread` | SharedArrayBuffer + Web Workers |

---

## Self-Assessment

### Q1: Compile a C++ library to .wasm with emcc and call it from JavaScript using cwrap

**Answer:**

**C++ source (math_lib.cpp):**

```cpp

#include <emscripten.h>
#include <cmath>

// EMSCRIPTEN_KEEPALIVE prevents dead code elimination
extern "C" {

EMSCRIPTEN_KEEPALIVE
int factorial(int n) {
    if (n <= 1) return 1;
    int result = 1;
    for (int i = 2; i <= n; ++i) result *= i;
    return result;
}

EMSCRIPTEN_KEEPALIVE
double distance(double x1, double y1, double x2, double y2) {
    double dx = x2 - x1, dy = y2 - y1;
    return std::sqrt(dx * dx + dy * dy);
}

EMSCRIPTEN_KEEPALIVE
int fibonacci(int n) {
    if (n <= 0) return 0;
    if (n == 1) return 1;
    int a = 0, b = 1;
    for (int i = 2; i <= n; ++i) {
        int tmp = a + b;
        a = b;
        b = tmp;
    }
    return b;
}

}  // extern "C"

```

**Compile:**

```bash

emcc math_lib.cpp -o math_lib.js \
    -sEXPORTED_FUNCTIONS='["_factorial","_distance","_fibonacci"]' \
    -sEXPORTED_RUNTIME_METHODS='["cwrap","ccall"]' \
    -O2
# Produces: math_lib.js + math_lib.wasm

```

**JavaScript usage:**

```html

<script src="math_lib.js"></script>
<script>
Module.onRuntimeInitialized = () => {
    // cwrap: create a reusable JS function wrapper
    const factorial = Module.cwrap('factorial', 'number', ['number']);
    const distance  = Module.cwrap('distance', 'number',
                                   ['number', 'number', 'number', 'number']);

    console.log(factorial(10));            // 3628800
    console.log(distance(0, 0, 3, 4));    // 5.0

    // ccall: one-shot call (no wrapper)
    const fib = Module.ccall('fibonacci', 'number', ['number'], [20]);
    console.log(fib);                      // 6765
};
</script>

```

### Q2: Use Embind to expose a C++ class as a JavaScript class with Emscripten

**Answer:**

```cpp

// vector2d.cpp
#include <emscripten/bind.h>
#include <cmath>
#include <string>

class Vector2D {
public:
    double x, y;

    Vector2D() : x(0), y(0) {}
    Vector2D(double x, double y) : x(x), y(y) {}

    double length() const { return std::sqrt(x * x + y * y); }

    Vector2D normalized() const {
        double len = length();
        if (len == 0) return {0, 0};
        return {x / len, y / len};
    }

    Vector2D operator+(const Vector2D& o) const { return {x + o.x, y + o.y}; }
    Vector2D operator*(double s) const { return {x * s, y * s}; }

    double dot(const Vector2D& o) const { return x * o.x + y * o.y; }

    std::string to_string() const {
        return "(" + std::to_string(x) + ", " + std::to_string(y) + ")";
    }
};

// ═══════════ Embind registration ═══════════
EMSCRIPTEN_BINDINGS(vector_module) {
    emscripten::class_<Vector2D>("Vector2D")
        .constructor<>()
        .constructor<double, double>()
        .property("x", &Vector2D::x)
        .property("y", &Vector2D::y)
        .function("length", &Vector2D::length)
        .function("normalized", &Vector2D::normalized)
        .function("add", &Vector2D::operator+)        // Rename operator
        .function("scale", &Vector2D::operator*)
        .function("dot", &Vector2D::dot)
        .function("toString", &Vector2D::to_string);
}

```

```bash

emcc vector2d.cpp -o vector2d.js --bind -O2

```

```javascript

// JavaScript — Vector2D is a real JS class!
Module.onRuntimeInitialized = () => {
    const v1 = new Module.Vector2D(3, 4);
    console.log(v1.length());       // 5.0
    console.log(v1.toString());     // "(3.000000, 4.000000)"

    const v2 = new Module.Vector2D(1, 0);
    const v3 = v1.add(v2);         // Vector2D(4, 4)
    console.log(v3.x, v3.y);      // 4, 4

    const norm = v1.normalized();
    console.log(norm.length());    // ~1.0

    // IMPORTANT: prevent memory leaks — delete C++ objects
    v1.delete();
    v2.delete();
    v3.delete();
    norm.delete();
};

```

### Q3: Explain the Asyncify transform and how it enables co_await-style async in Wasm C++

**Answer:**

Wasm executes **synchronously** — there's no native `sleep()` or blocking I/O. Asyncify solves this by transforming synchronous C++ code to pause and resume, enabling async JS calls from blocking C++ code.

```cpp

Without Asyncify:
  C++ calls sleep(1000) → ENTIRE browser tab freezes for 1s

With Asyncify:
  C++ calls sleep(1000) → Asyncify pauses C++ execution
                         → Returns control to JS event loop
                         → setTimeout(1000ms) fires
                         → Asyncify resumes C++ from where it paused

```

**How it works internally:**

```cpp

1. Compiler instruments every function call with save/restore state
2. When async operation triggered:

   ┌──────────────┐
   │ C++ function  │
   │ ...           │
   │ fetch(url)    │ ← triggers async
   │ ...           │
   └──────┬───────┘
          │ save stack + locals to memory
          ▼
   Return to JS event loop
          │ ... network request completes ...
          ▼
   Restore stack + locals from memory
   ┌──────────────┐
   │ C++ resumes   │
   │ at fetch()+1  │
   │ ...           │
   └──────────────┘

```

```cpp

// async_example.cpp
#include <emscripten.h>
#include <emscripten/fetch.h>
#include <cstdio>

// This C++ code looks synchronous, but Asyncify makes it non-blocking!
extern "C" EMSCRIPTEN_KEEPALIVE void run_demo() {
    printf("Step 1: starting\n");

    // emscripten_sleep pauses C++ but doesn't freeze the browser
    emscripten_sleep(2000);  // "sleep" 2 seconds (non-blocking)

    printf("Step 2: after sleep\n");

    // Fetch URL synchronously (Asyncify unwinding happens here)
    emscripten_fetch_attr_t attr;
    emscripten_fetch_attr_init(&attr);
    attr.attributes = EMSCRIPTEN_FETCH_LOAD_TO_MEMORY | EMSCRIPTEN_FETCH_SYNCHRONOUS;
    strcpy(attr.requestMethod, "GET");

    emscripten_fetch_t* fetch = emscripten_fetch(&attr, "https://api.example.com/data");
    printf("Step 3: got %llu bytes\n", fetch->numBytes);
    emscripten_fetch_close(fetch);
}

```

```bash

emcc async_example.cpp -o async.js \
    -sASYNCIFY \
    -sASYNCIFY_STACK_SIZE=65536 \
    -sFETCH \
    -O2

```

**Asyncify cost:** ~10% code size increase, ~5% runtime overhead for instrumented functions.

---

## Notes

- `emcc` is a drop-in replacement for `gcc`/`clang` — accepts same flags plus Emscripten-specific `-s` options
- Use `-O2` or `-O3` for production; `-g` for debugging with DWARF info
- Embind objects MUST be `.delete()`d from JS to avoid C++ memory leaks
- `-sALLOW_MEMORY_GROWTH` enables dynamic heap resizing (slight perf cost)
- Emscripten supports most of C++20 (std::format, ranges, concepts) via libc++
- For file I/O, Emscripten provides MEMFS (in-memory), IDBFS (IndexedDB), NODEFS (Node.js)
