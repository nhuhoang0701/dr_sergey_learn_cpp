# Use SWIG to generate bindings for multiple languages from one interface file

**Category:** Interoperability  
**Item:** #699  
**Reference:** <https://www.swig.org/>  

---

## Topic Overview

**SWIG (Simplified Wrapper and Interface Generator)** reads C/C++ header files and a `.i` interface file, then auto-generates binding code for Python, Java, C#, Ruby, Go, Lua, and 20+ other languages. One interface file → multiple language targets.

### SWIG Workflow

```cpp

                        ┌──────────────┐
mathlib.h ──────────────│              │──► mathlib_wrap.cxx (Python)
                        │    SWIG      │──► mathlib_wrap.cxx (Java)
mathlib.i ──────────────│  generator   │──► mathlib_wrap.cxx (C#)
(interface file)        │              │──► mathlib.py / MathLib.java / etc.
                        └──────────────┘
                               │
                        One .i file generates wrappers
                        for ALL target languages

```

### SWIG vs pybind11 vs nanobind

| Feature | SWIG | pybind11 | nanobind |
| --- | :---: | :---: | :---: |
| Multiple languages | **20+** | Python only | Python only |
| C++ class support | Good | **Excellent** | **Excellent** |
| Template support | Limited (%template) | **Full** | **Full** |
| Python-ness | Basic | **Pythonic** | **Pythonic** |
| C code support | **Excellent** | Good | Good |
| Maintenance effort | Low (auto-gen) | Medium | Medium |
| Binary size | Medium | Large | **Small** |
| learning curve | Moderate | Easy | Easy |

---

## Self-Assessment

### Q1: Write a SWIG .i interface file for a C++ class and generate Python bindings

**Answer:**

```cpp

// ═══════════ mathlib.h — C++ header ═══════════
#ifndef MATHLIB_H
#define MATHLIB_H

#include <vector>
#include <string>
#include <cmath>
#include <stdexcept>

class Matrix {
    std::vector<double> data_;
    int rows_, cols_;

public:
    Matrix(int rows, int cols);
    ~Matrix() = default;

    double get(int r, int c) const;
    void set(int r, int c, double val);
    int rows() const { return rows_; }
    int cols() const { return cols_; }

    Matrix multiply(const Matrix& other) const;
    double determinant() const;  // for 2x2

    std::string to_string() const;
};

// Free functions
double dot_product(const std::vector<double>& a, const std::vector<double>& b);
std::vector<double> linspace(double start, double end, int n);

#endif

```

```cpp

// ═══════════ mathlib.cpp — implementation ═══════════
#include "mathlib.h"
#include <sstream>
#include <numeric>

Matrix::Matrix(int rows, int cols) : data_(rows * cols, 0), rows_(rows), cols_(cols) {}

double Matrix::get(int r, int c) const {
    if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
        throw std::out_of_range("Index out of bounds");
    return data_[r * cols_ + c];
}

void Matrix::set(int r, int c, double val) {
    if (r < 0 || r >= rows_ || c < 0 || c >= cols_)
        throw std::out_of_range("Index out of bounds");
    data_[r * cols_ + c] = val;
}

Matrix Matrix::multiply(const Matrix& other) const {
    if (cols_ != other.rows_) throw std::invalid_argument("Shape mismatch");
    Matrix result(rows_, other.cols_);
    for (int i = 0; i < rows_; ++i)
        for (int j = 0; j < other.cols_; ++j) {
            double sum = 0;
            for (int k = 0; k < cols_; ++k)
                sum += get(i, k) * other.get(k, j);
            result.set(i, j, sum);
        }
    return result;
}

double Matrix::determinant() const {
    if (rows_ != 2 || cols_ != 2) throw std::runtime_error("Only 2x2 supported");
    return get(0,0)*get(1,1) - get(0,1)*get(1,0);
}

std::string Matrix::to_string() const {
    std::ostringstream oss;
    for (int i = 0; i < rows_; ++i) {
        for (int j = 0; j < cols_; ++j)
            oss << get(i, j) << " ";
        oss << "\n";
    }
    return oss.str();
}

double dot_product(const std::vector<double>& a, const std::vector<double>& b) {
    return std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
}

std::vector<double> linspace(double start, double end, int n) {
    std::vector<double> result(n);
    for (int i = 0; i < n; ++i)
        result[i] = start + (end - start) * i / (n - 1);
    return result;
}

```

```swig

// ═══════════ mathlib.i — SWIG interface file ═══════════
%module mathlib

%{
// This block is copied verbatim into the generated wrapper
#include "mathlib.h"
%}

// Enable STL support
%include "std_string.i"
%include "std_vector.i"

// Instantiate templates SWIG needs to wrap
%template(DoubleVector) std::vector<double>;

// Enable exception translation
%include "exception.i"
%exception {
    try {
        $action
    } catch (const std::out_of_range& e) {
        SWIG_exception(SWIG_IndexError, e.what());
    } catch (const std::invalid_argument& e) {
        SWIG_exception(SWIG_ValueError, e.what());
    } catch (const std::exception& e) {
        SWIG_exception(SWIG_RuntimeError, e.what());
    }
}

// Add Python __str__ method
%extend Matrix {
    std::string __str__() {
        return $self->to_string();
    }
    std::string __repr__() {
        return "<Matrix " + std::to_string($self->rows()) + "x" +
               std::to_string($self->cols()) + ">";
    }
}

// Parse the header — SWIG generates wrappers for everything in it
%include "mathlib.h"

```

```bash

# Generate Python bindings:
swig -c++ -python mathlib.i
# Produces: mathlib_wrap.cxx, mathlib.py

# Compile:
g++ -shared -fPIC -o _mathlib.so \
    mathlib.cpp mathlib_wrap.cxx \
    -I/usr/include/python3.11 \
    $(python3-config --ldflags)

# Generate Java bindings (same .i file!):
swig -c++ -java mathlib.i
# Produces: mathlib_wrap.cxx, Matrix.java, mathlib.java, etc

```

```python

# Python usage:
import mathlib

m = mathlib.Matrix(2, 2)
m.set(0, 0, 1); m.set(0, 1, 2)
m.set(1, 0, 3); m.set(1, 1, 4)
print(m)  # 1 2 \n 3 4
print(f"Det: {m.determinant()}")  # Det: -2.0

v = mathlib.linspace(0, 1, 5)
print(list(v))  # [0.0, 0.25, 0.5, 0.75, 1.0]

```

### Q2: Explain when SWIG is preferable to pybind11: multiple target languages, legacy codebase

**Answer:**

**Use SWIG when:**

| Scenario | Why SWIG wins |
| --- | --- |
| Need Python + Java + C# | One `.i` file → all targets |
| Large C API (100+ functions) | Auto-wraps entire headers with `%include` |
| Legacy C codebase | SWIG handles pure C naturally |
| Minimal maintenance budget | Auto-generated — update `.h`, re-run SWIG |
| Embedded scripting (Lua, Tcl) | SWIG supports niche languages |

**Use pybind11/nanobind when:**

| Scenario | Why pybind11 wins |
| --- | --- |
| Python-only target | More Pythonic API |
| Heavy template use | SWIG templates need explicit `%template` |
| Virtual method override in Python | Trampoline classes are natural |
| NumPy integration | First-class `py::array_t` support |
| Modern C++ (concepts, ranges) | Better C++17/20 support |

**SWIG limitations:**

- `%template` must be declared for every template instantiation you want to expose
- STL containers need `%include "std_vector.i"` etc. — must be explicit
- Generated Python code looks "C-like" not Pythonic (no properties by default)
- No direct NumPy buffer protocol — need typemaps
- Complex C++ (SFINAE, constexpr, concepts) may confuse SWIG parser

### Q3: Compare SWIG-generated bindings with manual pybind11 bindings for type safety

**Answer:**

| Aspect | SWIG | pybind11 |
| --- | --- | --- |
| **Type checking** | Runtime (proxy objects) | **Compile-time** (templates) |
| **Wrong types** | Silent conversion or crash | **TypeError with message** |
| **Overload resolution** | First-match (order-dependent) | **Best-match** |
| **Default args** | Supported | Supported + visible in help() |
| **Return by value** | Copy (usually) | Copy, move, or reference (configurable) |
| **Smart pointers** | `%shared_ptr` directive | **Automatic** (shared_ptr holder) |
| **Const correctness** | Mostly ignored | **Enforced** |

```python

# SWIG: type safety is weaker
import mathlib_swig
m = mathlib_swig.Matrix(2, 2)
m.set(0, 0, "hello")  # May silently pass or crash — no compile-time check
# SWIG relies on C-style conversions

# pybind11: strict type checking
import mathlib_pb11
m = mathlib_pb11.Matrix(2, 2)
m.set(0, 0, "hello")  # TypeError: incompatible function arguments
# pybind11 checks types at call time using C++ template matching

```

```cpp

SWIG approach:
  C++ header → SWIG parser → generated C code → compiled
  Type info: embedded in proxy classes at runtime
  Safety: depends on SWIG typemaps (customizable but manual)

pybind11 approach:
  C++ binding code → C++ compiler → compiled
  Type info: C++ template type system (checked at compile time)
  Safety: inherits C++ type system directly

```

---

## Notes

- SWIG is best for "wrap an entire C library in 5 minutes" scenarios
- Use `%rename` to give Python-friendly names: `%rename(__getitem__) Matrix::get;`
- `%extend` adds methods that didn't exist in C++ — useful for `__str__`, `__len__`, etc.
- SWIG 4.1+ supports C++17; C++20 support is partial
- For new projects targeting Python only, pybind11/nanobind is strongly preferred
- SWIG-generated code can be large — consider `-O` flag for optimization
