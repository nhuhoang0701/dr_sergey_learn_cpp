# Use ctypes / cffi from Python to call C++ shared libraries

**Category:** Interoperability  
**Item:** #778  
**Reference:** <https://docs.python.org/3/library/ctypes.html>  

---

## Topic Overview

`ctypes` and `cffi` let Python call functions in compiled C/C++ shared libraries (`.so` / `.dll`) without any compilation step on the Python side. The C++ library must export functions with `extern "C"` linkage.

### Call Flow

```cpp

Python Process
┌─────────────────────────────────────┐
│ import ctypes                       │
│ lib = ctypes.CDLL("./libmath.so")   │──► dlopen / LoadLibrary
│ lib.add.restype = ctypes.c_int      │
│ result = lib.add(3, 4)              │──► FFI call to C symbol "add"
│ print(result)  # 7                  │
└─────────────────────────────────────┘
         │
         ▼
C++ Shared Library (libmath.so)
┌─────────────────────────────────────┐
│ extern "C" int add(int a, int b) {  │
│     return a + b;                   │
│ }                                   │
└─────────────────────────────────────┘

```

### ctypes vs cffi vs pybind11

| Feature | ctypes | cffi | pybind11/nanobind |
| --- | :---: | :---: | :---: |
| Install needed | No (stdlib) | Yes (pip) | Yes (pip + compiler) |
| C++ class support | No | No | **Yes** |
| ABI mode (no compile) | Yes | Yes | No |
| API mode (compile) | No | Yes | Yes |
| Performance | Good | Better | **Best** |
| Type checking | Manual | Auto from headers | **Auto** |
| Exception handling | No | No | **Yes** |

---

## Self-Assessment

### Q1: Export a C-linkage function from a C++ shared library and call it from Python using ctypes

**Answer:**

```cpp

// ═══════════ mathlib.cpp — C++ shared library ═══════════
#include <cmath>
#include <cstring>

extern "C" {

// Simple arithmetic
int add(int a, int b) {
    return a + b;
}

double compute_distance(double x1, double y1, double x2, double y2) {
    double dx = x2 - x1;
    double dy = y2 - y1;
    return std::sqrt(dx * dx + dy * dy);
}

// String processing — return value via output parameter
int string_reverse(const char* input, char* output, int max_len) {
    if (!input || !output) return -1;
    int len = static_cast<int>(std::strlen(input));
    if (len >= max_len) return -2;

    for (int i = 0; i < len; ++i) {
        output[i] = input[len - 1 - i];
    }
    output[len] = '\0';
    return 0;
}

// Array processing
double array_sum(const double* arr, int n) {
    double sum = 0.0;
    for (int i = 0; i < n; ++i) sum += arr[i];
    return sum;
}

}  // extern "C"

```

```bash

# Build shared library
# Linux:
g++ -shared -fPIC -o libmath.so mathlib.cpp
# Windows:
# cl /LD mathlib.cpp /Fe:mathlib.dll
# macOS:
# g++ -shared -o libmath.dylib mathlib.cpp

```

```python

# ═══════════ use_mathlib.py — Python consumer ═══════════
import ctypes
import os

# Load the shared library
if os.name == 'nt':
    lib = ctypes.CDLL("./mathlib.dll")
else:
    lib = ctypes.CDLL("./libmath.so")

# ─── Set argument and return types (critical for correctness!) ───
lib.add.argtypes = [ctypes.c_int, ctypes.c_int]
lib.add.restype = ctypes.c_int

lib.compute_distance.argtypes = [
    ctypes.c_double, ctypes.c_double,
    ctypes.c_double, ctypes.c_double
]
lib.compute_distance.restype = ctypes.c_double

lib.array_sum.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int]
lib.array_sum.restype = ctypes.c_double

# ─── Call functions ───
print(lib.add(3, 4))                    # Output: 7
print(lib.compute_distance(0, 0, 3, 4)) # Output: 5.0

# ─── Pass an array ───
data = [1.0, 2.0, 3.0, 4.0, 5.0]
arr = (ctypes.c_double * len(data))(*data)  # Create C array
print(lib.array_sum(arr, len(data)))         # Output: 15.0

# ─── String function ───
lib.string_reverse.argtypes = [ctypes.c_char_p, ctypes.c_char_p, ctypes.c_int]
lib.string_reverse.restype = ctypes.c_int

output = ctypes.create_string_buffer(256)
lib.string_reverse(b"Hello", output, 256)
print(output.value)  # Output: b'olleH'

```

### Q2: Show how to pass structs between Python and C++ using ctypes Structure

**Answer:**

```cpp

// ═══════════ geometry.cpp — C++ struct-based API ═══════════
#include <cmath>

struct Point {
    double x;
    double y;
};

struct Rect {
    Point top_left;
    Point bottom_right;
};

struct Stats {
    double min_val;
    double max_val;
    double mean;
    int count;
};

extern "C" {

double point_distance(const Point* a, const Point* b) {
    double dx = b->x - a->x;
    double dy = b->y - a->y;
    return std::sqrt(dx * dx + dy * dy);
}

double rect_area(const Rect* r) {
    double w = std::abs(r->bottom_right.x - r->top_left.x);
    double h = std::abs(r->bottom_right.y - r->top_left.y);
    return w * h;
}

// Return struct by filling output parameter
void compute_stats(const double* data, int n, Stats* out) {
    if (n <= 0) return;
    out->min_val = data[0];
    out->max_val = data[0];
    double sum = 0;
    for (int i = 0; i < n; ++i) {
        if (data[i] < out->min_val) out->min_val = data[i];
        if (data[i] > out->max_val) out->max_val = data[i];
        sum += data[i];
    }
    out->mean = sum / n;
    out->count = n;
}

}  // extern "C"

```

```python

# ═══════════ use_geometry.py ═══════════
import ctypes

lib = ctypes.CDLL("./libgeometry.so")

# ─── Define matching Python structures ───
class Point(ctypes.Structure):
    _fields_ = [
        ("x", ctypes.c_double),
        ("y", ctypes.c_double),
    ]

class Rect(ctypes.Structure):
    _fields_ = [
        ("top_left", Point),        # Nested struct
        ("bottom_right", Point),
    ]

class Stats(ctypes.Structure):
    _fields_ = [
        ("min_val", ctypes.c_double),
        ("max_val", ctypes.c_double),
        ("mean", ctypes.c_double),
        ("count", ctypes.c_int),
    ]

# ─── Set types ───
lib.point_distance.argtypes = [ctypes.POINTER(Point), ctypes.POINTER(Point)]
lib.point_distance.restype = ctypes.c_double

lib.rect_area.argtypes = [ctypes.POINTER(Rect)]
lib.rect_area.restype = ctypes.c_double

lib.compute_stats.argtypes = [
    ctypes.POINTER(ctypes.c_double), ctypes.c_int, ctypes.POINTER(Stats)
]
lib.compute_stats.restype = None

# ─── Use structs ───
a = Point(0.0, 0.0)
b = Point(3.0, 4.0)
print(f"Distance: {lib.point_distance(a, b)}")  # Output: Distance: 5.0

r = Rect(Point(0, 0), Point(10, 5))
print(f"Area: {lib.rect_area(r)}")  # Output: Area: 50.0

# ─── Output parameter pattern ───
data = [3.0, 1.0, 4.0, 1.0, 5.0, 9.0]
arr = (ctypes.c_double * len(data))(*data)
stats = Stats()
lib.compute_stats(arr, len(data), ctypes.byref(stats))
print(f"Min: {stats.min_val}, Max: {stats.max_val}, "
      f"Mean: {stats.mean:.2f}, Count: {stats.count}")
# Output: Min: 1.0, Max: 9.0, Mean: 3.83, Count: 6

```

### Q3: Explain when ctypes is sufficient vs when pybind11 or nanobind is needed

**Answer:**

| Scenario | ctypes/cffi | pybind11/nanobind |
| --- | :---: | :---: |
| Call a few C functions | **Yes** | Overkill |
| Use existing `.so`/`.dll` | **Yes** | Needs rebuild |
| Wrap C++ classes | Painful | **Yes** |
| Handle C++ exceptions | No | **Yes** |
| Template functions | No | **Yes** |
| NumPy array interop | Manual | **Yes** (buffer protocol) |
| Callbacks Python→C++ | Fragile | **Yes** |
| Performance-critical tight loop | Good | **Better** (less overhead) |
| No build step needed | **Yes** | No |

**Use ctypes when:**

- You need to call a small number of C-style functions from an existing library
- The library already has a C API (e.g., SQLite, OpenSSL, system libraries)
- You want zero compile dependencies on the Python side

**Use pybind11/nanobind when:**

- You're wrapping C++ classes with methods, inheritance, virtual functions
- You need automatic type conversion (std::string ↔ str, std::vector ↔ list)
- You want exceptions to propagate properly
- You're building a Python package that includes C++ code

```python

# ctypes: manual, error-prone, but zero dependencies
lib = ctypes.CDLL("./libstats.so")
lib.mean.argtypes = [ctypes.POINTER(ctypes.c_double), ctypes.c_int]
lib.mean.restype = ctypes.c_double
arr = (ctypes.c_double * 3)(1.0, 2.0, 3.0)
result = lib.mean(arr, 3)
# Forget to set argtypes? Silent wrong answer or segfault

```

```python

# pybind11: automatic, safe, but requires compilation
import stats  # compiled C++ extension module
result = stats.mean([1.0, 2.0, 3.0])  # accepts Python list directly
# Wrong type? Raises TypeError with clear message

```

---

## Notes

- Always set `argtypes` and `restype` — without them, ctypes assumes `c_int` for everything
- `ctypes.byref(x)` passes a pointer (faster than `ctypes.pointer(x)`)
- cffi's API mode precompiles the wrapper → faster startup than ctypes
- Struct padding differences can cause subtle bugs — use `#pragma pack` or `__attribute__((packed))` to match
- On Windows, use `CDLL` for `__cdecl` functions, `WinDLL` for `__stdcall` (Win32 API)
- Memory allocated in C++ must be freed in C++ — never `free()` from Python side
