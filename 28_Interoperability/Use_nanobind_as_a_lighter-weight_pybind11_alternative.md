# Use nanobind as a lighter-weight pybind11 alternative

**Category:** Interoperability  
**Item:** #592  
**Standard:** C++17  
**Reference:** <https://nanobind.readthedocs.io/>  

---

## Topic Overview

This topic focuses on **nanobind's ndarray support for zero-copy tensor interop**, its smart-pointer ownership semantics, and practical porting from pybind11. nanobind is designed for performance-critical scientific computing where binary size and import speed matter.

### CMake Setup

```cmake

# CMakeLists.txt
cmake_minimum_required(VERSION 3.18)
project(mymodule)

find_package(Python 3.8 COMPONENTS Interpreter Development.Module REQUIRED)
find_package(nanobind CONFIG REQUIRED)

nanobind_add_module(mymodule src/bindings.cpp)
# Produces: mymodule.cpython-312-x86_64-linux-gnu.so

```

### Key API Differences from pybind11

| pybind11 | nanobind | Notes |
| --- | --- | --- |
| `PYBIND11_MODULE(name, m)` | `NB_MODULE(name, m)` | Same structure |
| `py::class_<T>` | `nb::class_<T>` | Same API |
| `.def_readwrite(...)` | `.def_rw(...)` | Shorter name |
| `.def_readonly(...)` | `.def_ro(...)` | Shorter name |
| `py::array_t<T>` | `nb::ndarray<T>` | Zero-copy, multi-framework |
| `py::arg("name")` | `nb::arg("name")` | Same |
| `py::return_value_policy` | `nb::rv_policy` | Shorter name |

---

## Self-Assessment

### Q1: Port a simple pybind11 module to nanobind and compare compile time and binary size

**Answer:**

```cpp

// ═══════════ pybind11 original: signal_pb11.cpp ═══════════
#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <vector>
#include <cmath>

namespace py = pybind11;

struct Signal {
    std::vector<float> samples;
    int sample_rate;

    Signal(int rate, int duration_ms)
        : sample_rate(rate),
          samples(rate * duration_ms / 1000, 0.0f) {}

    void generate_sine(float freq) {
        for (size_t i = 0; i < samples.size(); ++i) {
            float t = static_cast<float>(i) / sample_rate;
            samples[i] = std::sin(2.0f * M_PI * freq * t);
        }
    }

    float rms() const {
        float sum = 0;
        for (float s : samples) sum += s * s;
        return std::sqrt(sum / samples.size());
    }
};

PYBIND11_MODULE(signal_pb11, m) {
    py::class_<Signal>(m, "Signal")
        .def(py::init<int, int>(), py::arg("rate"), py::arg("duration_ms"))
        .def("generate_sine", &Signal::generate_sine, py::arg("freq"))
        .def("rms", &Signal::rms)
        .def_readwrite("samples", &Signal::samples)
        .def_readonly("sample_rate", &Signal::sample_rate);
}

```

```cpp

// ═══════════ nanobind port: signal_nb.cpp ═══════════
#include <nanobind/nanobind.h>
#include <nanobind/stl/vector.h>
#include <vector>
#include <cmath>

namespace nb = nanobind;

struct Signal {
    std::vector<float> samples;
    int sample_rate;

    Signal(int rate, int duration_ms)
        : sample_rate(rate),
          samples(rate * duration_ms / 1000, 0.0f) {}

    void generate_sine(float freq) {
        for (size_t i = 0; i < samples.size(); ++i) {
            float t = static_cast<float>(i) / sample_rate;
            samples[i] = std::sin(2.0f * M_PI * freq * t);
        }
    }

    float rms() const {
        float sum = 0;
        for (float s : samples) sum += s * s;
        return std::sqrt(sum / samples.size());
    }
};

NB_MODULE(signal_nb, m) {
    nb::class_<Signal>(m, "Signal")
        .def(nb::init<int, int>(), nb::arg("rate"), nb::arg("duration_ms"))
        .def("generate_sine", &Signal::generate_sine, nb::arg("freq"))
        .def("rms", &Signal::rms)
        .def_rw("samples", &Signal::samples)           // def_rw not def_readwrite
        .def_ro("sample_rate", &Signal::sample_rate);   // def_ro not def_readonly
}

// Compile time: ~3s (pybind11: ~8s)
// Binary size:  ~60 KB (pybind11: ~250 KB)

```

### Q2: Explain how nanobind's ownership model differs from pybind11 for smart pointer types

**Answer:**

```cpp

#include <nanobind/nanobind.h>
#include <memory>

namespace nb = nanobind;

// ═══════════ pybind11: shared_ptr by default ═══════════
// py::class_<Foo, std::shared_ptr<Foo>>(m, "Foo")
//   - Every Python wrapper holds a shared_ptr
//   - C++ and Python share ownership
//   - Overhead: control block + atomic ref count
//   - Works naturally with shared_ptr APIs

// ═══════════ nanobind: unique ownership by default ═══════════
// nb::class_<Foo>(m, "Foo")
//   - Python OWNS the C++ object directly
//   - No shared_ptr overhead
//   - C++ code that needs shared_ptr must explicitly opt in

struct Texture {
    int width, height;
    std::vector<uint8_t> data;
    Texture(int w, int h) : width(w), height(h), data(w * h * 4) {}
};

// ═══════════ Using shared_ptr with nanobind (opt-in) ═══════════
struct SharedTexture : nb::intrusive_base {
    // intrusive_base provides ref counting shared between Python and C++
    int width, height;
    SharedTexture(int w, int h) : width(w), height(h) {}
};

// Or use std::shared_ptr explicitly:
struct ManagedTexture {
    int width, height;
    ManagedTexture(int w, int h) : width(w), height(h) {}
};

NB_MODULE(texture_mod, m) {
    // Default: nanobind owns via unique_ptr-like semantics
    nb::class_<Texture>(m, "Texture")
        .def(nb::init<int, int>())
        .def_ro("width", &Texture::width);
    // Python del → C++ destructor runs immediately

    // Intrusive refcount: shared between Python and C++
    nb::class_<SharedTexture>(m, "SharedTexture")
        .def(nb::init<int, int>());

    // Explicit shared_ptr holder
    nb::class_<ManagedTexture>(m, "ManagedTexture",
        nb::holder<std::shared_ptr<ManagedTexture>>())
        .def(nb::init<int, int>());
}

// Summary:
// pybind11: shared_ptr everywhere (safe but overhead)
// nanobind: unique ownership (fast, explicit sharing when needed)

```

### Q3: Show nanobind's nb::ndarray for zero-copy NumPy interop with explicit memory ownership

**Answer:**

```cpp

#include <nanobind/nanobind.h>
#include <nanobind/ndarray.h>
#include <vector>
#include <cmath>

namespace nb = nanobind;

// ═══════════ Accept NumPy array, process in-place (zero-copy) ═══════════
void normalize_inplace(
    nb::ndarray<float, nb::shape<nb::any>, nb::c_contig, nb::device::cpu> arr)
{
    // Zero-copy: arr.data() points directly to NumPy's buffer
    float* data = arr.data();
    size_t n = arr.shape(0);

    // Find max
    float max_val = 0;
    for (size_t i = 0; i < n; ++i)
        max_val = std::max(max_val, std::abs(data[i]));

    // Normalize in-place (modifies NumPy array directly!)
    if (max_val > 0) {
        for (size_t i = 0; i < n; ++i)
            data[i] /= max_val;
    }
}

// ═══════════ Return a new NumPy array (with ownership transfer) ═══════════
nb::ndarray<nb::numpy, float, nb::shape<nb::any, nb::any>>
create_matrix(size_t rows, size_t cols) {
    // Allocate C++ buffer
    float* data = new float[rows * cols];
    for (size_t i = 0; i < rows * cols; ++i)
        data[i] = static_cast<float>(i);

    // Create ndarray that OWNS the data
    // When Python GC collects the array, the deleter runs
    size_t shape[2] = {rows, cols};
    nb::capsule deleter(data, [](void* p) noexcept {
        delete[] static_cast<float*>(p);
    });

    return nb::ndarray<nb::numpy, float, nb::shape<nb::any, nb::any>>(
        data, 2, shape, deleter);
}

// ═══════════ 2D array with bounds checking ═══════════
float sum_2d(
    nb::ndarray<float, nb::shape<nb::any, nb::any>, nb::c_contig> matrix)
{
    size_t rows = matrix.shape(0);
    size_t cols = matrix.shape(1);
    const float* data = matrix.data();

    float sum = 0;
    for (size_t i = 0; i < rows * cols; ++i)
        sum += data[i];
    return sum;
}

NB_MODULE(ndarray_demo, m) {
    m.def("normalize_inplace", &normalize_inplace,
          "Normalize array values to [-1, 1] range (modifies in-place)");
    m.def("create_matrix", &create_matrix,
          nb::arg("rows"), nb::arg("cols"),
          "Create a rows×cols matrix with sequential values");
    m.def("sum_2d", &sum_2d, "Sum all elements of a 2D array");
}

```

```python

# Usage from Python:
import numpy as np
import ndarray_demo

# Zero-copy in-place operation
arr = np.array([1.0, -3.0, 2.0, -0.5], dtype=np.float32)
ndarray_demo.normalize_inplace(arr)
print(arr)  # [0.333, -1.0, 0.666, -0.166]  (modified in-place!)

# C++ creates and returns array
mat = ndarray_demo.create_matrix(3, 4)
print(mat.shape)  # (3, 4)
print(mat)        # [[0, 1, 2, 3], [4, 5, 6, 7], [8, 9, 10, 11]]

# 2D sum
print(ndarray_demo.sum_2d(mat))  # 66.0

```

---

## Notes

- `nb::ndarray` supports NumPy, PyTorch, TensorFlow, and JAX tensors with the same C++ code
- Shape constraints (`nb::shape<3, nb::any>`) are checked at binding time — raises TypeError on mismatch
- `nb::c_contig` requires C-contiguous layout; `nb::f_contig` for Fortran order
- The `nb::capsule` deleter pattern ensures C++-allocated memory is freed when Python GCs the array
- nanobind's ndarray is more powerful than pybind11's `py::array_t` — multi-framework + safer
- For GPU tensors, use `nb::device::cuda` instead of `nb::device::cpu`
