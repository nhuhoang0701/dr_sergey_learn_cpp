# Use pybind11 for C++/Python interoperability

**Category:** Interoperability  
**Item:** #772  
**Standard:** C++11  
**Reference:** <https://pybind11.readthedocs.io>  

---

## Topic Overview

**pybind11** is a lightweight header-only library that creates Python bindings for C++ code. It enables exposing C++ classes, functions, and data structures to Python with minimal boilerplate. pybind11 handles type conversions, exceptions, and NumPy arrays automatically.

### pybind11 Architecture

```cpp

Python                          C++ Extension Module (.so/.pyd)
┌──────────────┐               ┌──────────────────────────────┐
│ import mymod │──dlopen──────►│ PYBIND11_MODULE(mymod, m) {  │
│              │               │   m.def("func", &func);      │
│ mymod.func() │──type check──►│   py::class_<Foo>(m,"Foo")   │
│              │               │     .def("method", &method);  │
│ result back  │◄──convert─────│ }                             │
└──────────────┘               └──────────────────────────────┘
                                        │
                                pybind11 auto-converts:
                                  str ↔ std::string
                                  list ↔ std::vector
                                  dict ↔ std::map
                                  int ↔ int/long
                                  float ↔ double
                                  numpy.ndarray ↔ py::array_t

```

---

## Self-Assessment

### Q1: Expose a C++ class to Python with pybind11: constructors, methods, and properties

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <pybind11/stl.h>  // std::vector, std::string conversions
#include <string>
#include <vector>
#include <cmath>
#include <stdexcept>

namespace py = pybind11;

// ═══════════ C++ class to expose ═══════════
class DataFrame {
    std::string name_;
    std::vector<std::string> columns_;
    std::vector<std::vector<double>> data_;

public:
    DataFrame(const std::string& name) : name_(name) {}

    void add_column(const std::string& col, const std::vector<double>& values) {
        columns_.push_back(col);
        data_.push_back(values);
    }

    size_t num_rows() const {
        return data_.empty() ? 0 : data_[0].size();
    }

    size_t num_columns() const { return columns_.size(); }

    double mean(const std::string& col) const {
        int idx = find_column(col);
        if (idx < 0) throw std::invalid_argument("Column not found: " + col);
        const auto& v = data_[idx];
        double sum = 0;
        for (double x : v) sum += x;
        return sum / v.size();
    }

    // Getters/setters for property binding
    const std::string& name() const { return name_; }
    void set_name(const std::string& n) { name_ = n; }
    const std::vector<std::string>& columns() const { return columns_; }

private:
    int find_column(const std::string& col) const {
        for (size_t i = 0; i < columns_.size(); ++i)
            if (columns_[i] == col) return static_cast<int>(i);
        return -1;
    }
};

// ═══════════ Module definition ═══════════
PYBIND11_MODULE(dataframe, m) {
    m.doc() = "Simple DataFrame implemented in C++";

    py::class_<DataFrame>(m, "DataFrame")
        // Constructor
        .def(py::init<const std::string&>(), py::arg("name"))

        // Methods
        .def("add_column", &DataFrame::add_column,
             py::arg("column"), py::arg("values"),
             "Add a named column with data")
        .def("mean", &DataFrame::mean, py::arg("column"),
             "Compute mean of a column")

        // Read-write property (getter + setter)
        .def_property("name", &DataFrame::name, &DataFrame::set_name)

        // Read-only properties
        .def_property_readonly("num_rows", &DataFrame::num_rows)
        .def_property_readonly("num_columns", &DataFrame::num_columns)
        .def_property_readonly("columns", &DataFrame::columns)

        // Special methods
        .def("__repr__", [](const DataFrame& df) {
            return "<DataFrame '" + df.name() + "' " +
                   std::to_string(df.num_rows()) + "x" +
                   std::to_string(df.num_columns()) + ">";
        })
        .def("__len__", &DataFrame::num_rows);
}

```

```python

# Python usage:
from dataframe import DataFrame

df = DataFrame("sales")
df.add_column("revenue", [100, 200, 300, 400])
df.add_column("cost", [50, 80, 120, 160])

print(df)            # <DataFrame 'sales' 4x2>
print(df.name)       # sales
df.name = "Q1 Sales"
print(len(df))       # 4
print(df.num_columns) # 2
print(df.columns)     # ['revenue', 'cost']
print(df.mean("revenue"))  # 250.0

```

### Q2: Handle NumPy array interop using pybind11::array_t for zero-copy buffer access

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
#include <cmath>
#include <algorithm>

namespace py = pybind11;

// ═══════════ Read-only access (no copy) ═══════════
double array_sum(py::array_t<double> input) {
    // Request read-only buffer
    auto buf = input.unchecked<1>();  // 1D array, no bounds checking
    double sum = 0;
    for (py::ssize_t i = 0; i < buf.shape(0); ++i)
        sum += buf(i);
    return sum;
}

// ═══════════ In-place modification (zero-copy) ═══════════
void normalize(py::array_t<double, py::array::c_style> arr) {
    // Mutable access — modifies NumPy array directly
    auto buf = arr.mutable_unchecked<1>();
    double max_val = 0;
    for (py::ssize_t i = 0; i < buf.shape(0); ++i)
        max_val = std::max(max_val, std::abs(buf(i)));
    if (max_val > 0) {
        for (py::ssize_t i = 0; i < buf.shape(0); ++i)
            buf(i) /= max_val;
    }
}

// ═══════════ 2D array: matrix multiplication ═══════════
py::array_t<double> matmul(py::array_t<double> a, py::array_t<double> b) {
    auto bufA = a.unchecked<2>();
    auto bufB = b.unchecked<2>();

    if (bufA.shape(1) != bufB.shape(0))
        throw std::runtime_error("Shape mismatch");

    auto M = bufA.shape(0), K = bufA.shape(1), N = bufB.shape(1);

    // Create output array
    py::array_t<double> result({M, N});
    auto bufR = result.mutable_unchecked<2>();

    for (py::ssize_t i = 0; i < M; ++i)
        for (py::ssize_t j = 0; j < N; ++j) {
            double sum = 0;
            for (py::ssize_t k = 0; k < K; ++k)
                sum += bufA(i, k) * bufB(k, j);
            bufR(i, j) = sum;
        }

    return result;
}

// ═══════════ Return new array with capsule ownership ═══════════
py::array_t<float> create_signal(int samples, float freq) {
    auto result = py::array_t<float>(samples);
    auto buf = result.mutable_unchecked<1>();
    for (py::ssize_t i = 0; i < samples; ++i) {
        float t = static_cast<float>(i) / 44100.0f;
        buf(i) = std::sin(2.0f * M_PI * freq * t);
    }
    return result;
}

PYBIND11_MODULE(numpy_demo, m) {
    m.def("array_sum", &array_sum, "Sum of 1D array (zero-copy read)");
    m.def("normalize", &normalize, "Normalize in-place (zero-copy write)");
    m.def("matmul", &matmul, "Matrix multiply (creates new array)");
    m.def("create_signal", &create_signal, "Generate sine wave signal");
}

```

### Q3: Show how pybind11 translates C++ exceptions to Python exceptions automatically

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <stdexcept>
#include <string>

namespace py = pybind11;

// ═══════════ Custom C++ exception ═══════════
class DatabaseError : public std::runtime_error {
public:
    int error_code;
    DatabaseError(const std::string& msg, int code)
        : std::runtime_error(msg), error_code(code) {}
};

// Functions that throw
double safe_divide(double a, double b) {
    if (b == 0.0) throw std::invalid_argument("Division by zero");
    return a / b;
}

std::string lookup(const std::string& key) {
    if (key.empty())
        throw std::invalid_argument("Empty key");
    if (key == "missing")
        throw std::out_of_range("Key not found: " + key);
    if (key == "db_error")
        throw DatabaseError("Connection lost", 503);
    return "value_for_" + key;
}

PYBIND11_MODULE(exception_demo, m) {
    // ═══════════ Auto-translation (built-in) ═══════════
    // pybind11 automatically translates:
    //   std::runtime_error    → RuntimeError
    //   std::invalid_argument → ValueError
    //   std::out_of_range     → IndexError
    //   std::domain_error     → ValueError
    //   std::overflow_error   → OverflowError
    //   std::bad_alloc        → MemoryError

    m.def("safe_divide", &safe_divide);
    m.def("lookup", &lookup);
    // safe_divide(1, 0) → Python ValueError: Division by zero
    // lookup("missing") → Python IndexError: Key not found: missing

    // ═══════════ Custom exception registration ═══════════
    // Create a Python exception class for DatabaseError
    static py::exception<DatabaseError> pyDbError(m, "DatabaseError");

    // Optional: add custom attributes to the Python exception
    py::register_exception_translator([](std::exception_ptr p) {
        try {
            if (p) std::rethrow_exception(p);
        } catch (const DatabaseError& e) {
            // Create Python exception with custom attribute
            py::object exc = pyDbError(e.what());
            exc.attr("error_code") = e.error_code;
            PyErr_SetObject(pyDbError.ptr(), exc.ptr());
        }
    });
}

```

```python

# Python usage:
from exception_demo import safe_divide, lookup, DatabaseError

try:
    safe_divide(1, 0)
except ValueError as e:
    print(f"Caught: {e}")  # Caught: Division by zero

try:
    lookup("missing")
except IndexError as e:
    print(f"Caught: {e}")  # Caught: Key not found: missing

try:
    lookup("db_error")
except DatabaseError as e:
    print(f"DB error: {e}, code={e.error_code}")
    # DB error: Connection lost, code=503

```

---

## Notes

- Install: `pip install pybind11` (Python) or `find_package(pybind11)` (CMake)
- `py::array::c_style` flag ensures C-contiguous layout — avoids silent copy on Fortran-ordered arrays
- `unchecked<N>()` skips bounds checking — faster but unsafe; use `at()` for safety
- Return value policies: `reference_internal` for member references, `copy` for small objects
- pybind11 uses `shared_ptr` by default for object ownership — more overhead than nanobind
- For new projects, consider nanobind for better performance; pybind11 for broader C++ standard support (C++11)
