# Use pybind11 to expose C++ classes and functions to Python

**Category:** Interoperability  
**Item:** #591  
**Reference:** <https://pybind11.readthedocs.io/>  

---

## Topic Overview

This topic covers **advanced pybind11 patterns**: `py::class_<T>` with inheritance and virtual functions, `py::register_exception` for custom exception types, and `py::buffer_protocol` to expose C++ containers as NumPy arrays without copying data.

### Key Binding Patterns

| Pattern | API | Use Case |
| --- | --- | --- |
| Constructor | `.def(py::init<args>())` | Create from Python |
| Method | `.def("name", &T::method)` | Regular methods |
| Static method | `.def_static("name", &T::func)` | Class methods |
| Property (R/W) | `.def_property("n", get, set)` | Getter + setter |
| Property (RO) | `.def_property_readonly("n", get)` | Read-only access |
| Operator | `.def(py::self + py::self)` | `__add__`, `__mul__`, etc. |
| Inheritance | `py::class_<D, B>(m, "D")` | Base class bindings |
| Virtual | `class PyB : B { ... }` | Override from Python |
| Buffer | `.def_buffer(...)` | NumPy zero-copy |

---

## Self-Assessment

### Q1: Expose a C++ class with py::class_<T> including constructors, methods, and properties

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <pybind11/stl.h>
#include <pybind11/operators.h>
#include <string>
#include <vector>
#include <sstream>
#include <cmath>

namespace py = pybind11;

// ═══════════ C++ class hierarchy ═══════════
class Shape {
public:
    std::string name;
    Shape(const std::string& n) : name(n) {}
    virtual ~Shape() = default;
    virtual double area() const = 0;
    virtual std::string describe() const {
        return name + " (area=" + std::to_string(area()) + ")";
    }
};

class Circle : public Shape {
    double radius_;
public:
    Circle(double r) : Shape("Circle"), radius_(r) {
        if (r < 0) throw std::invalid_argument("Negative radius");
    }
    double area() const override { return M_PI * radius_ * radius_; }
    double radius() const { return radius_; }
    void set_radius(double r) { radius_ = r; }
};

class Rectangle : public Shape {
public:
    double width, height;
    Rectangle(double w, double h) : Shape("Rectangle"), width(w), height(h) {}
    double area() const override { return width * height; }
    double perimeter() const { return 2 * (width + height); }

    // Operator support
    bool operator==(const Rectangle& other) const {
        return width == other.width && height == other.height;
    }
};

// ═══════════ Trampoline class for virtual overrides in Python ═══════════
class PyShape : public Shape {
public:
    using Shape::Shape;  // Inherit constructors

    double area() const override {
        PYBIND11_OVERRIDE_PURE(double, Shape, area);
    }

    std::string describe() const override {
        PYBIND11_OVERRIDE(std::string, Shape, describe);
    }
};

// ═══════════ Module ═══════════
PYBIND11_MODULE(shapes, m) {
    // Base class with trampoline
    py::class_<Shape, PyShape>(m, "Shape")
        .def(py::init<const std::string&>())
        .def("area", &Shape::area)
        .def("describe", &Shape::describe)
        .def_readwrite("name", &Shape::name);

    // Derived: Circle
    py::class_<Circle, Shape>(m, "Circle")
        .def(py::init<double>(), py::arg("radius"))
        .def("area", &Circle::area)
        .def_property("radius", &Circle::radius, &Circle::set_radius);

    // Derived: Rectangle with operators
    py::class_<Rectangle, Shape>(m, "Rectangle")
        .def(py::init<double, double>(), py::arg("width"), py::arg("height"))
        .def("area", &Rectangle::area)
        .def("perimeter", &Rectangle::perimeter)
        .def_readwrite("width", &Rectangle::width)
        .def_readwrite("height", &Rectangle::height)
        .def(py::self == py::self)  // __eq__
        .def("__repr__", [](const Rectangle& r) {
            return "Rectangle(" + std::to_string(r.width) + ", " +
                   std::to_string(r.height) + ")";
        });

    // Free function that accepts base class
    m.def("total_area", [](const std::vector<Shape*>& shapes) {
        double total = 0;
        for (auto* s : shapes) total += s->area();
        return total;
    });
}

```

```python

# Python usage — including overriding virtual from Python:
from shapes import Shape, Circle, Rectangle

c = Circle(5.0)
print(c.area())      # 78.539...
c.radius = 10.0
print(c.area())      # 314.159...

r = Rectangle(3, 4)
print(r.perimeter())  # 14.0
print(r == Rectangle(3, 4))  # True

# Override virtual method from Python
class Triangle(Shape):
    def __init__(self, base, height):
        super().__init__("Triangle")
        self.base = base
        self.height = height

    def area(self):
        return 0.5 * self.base * self.height

t = Triangle(6, 8)
print(t.area())       # 24.0
print(t.describe())   # Triangle (area=24.0)

```

### Q2: Handle C++ exceptions by translating them to Python exceptions using py::register_exception

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <stdexcept>

namespace py = pybind11;

// ═══════════ Custom exception hierarchy ═══════════
class AppError : public std::runtime_error {
public:
    using std::runtime_error::runtime_error;
};

class ValidationError : public AppError {
public:
    std::string field;
    ValidationError(const std::string& f, const std::string& msg)
        : AppError(msg), field(f) {}
};

class NotFoundError : public AppError {
public:
    int code;
    NotFoundError(const std::string& msg, int c)
        : AppError(msg), code(c) {}
};

void validate_email(const std::string& email) {
    if (email.find('@') == std::string::npos)
        throw ValidationError("email", "Invalid email: " + email);
}

void find_user(int id) {
    throw NotFoundError("User not found", 404);
}

PYBIND11_MODULE(exceptions_demo, m) {
    // ═══════════ Register exception hierarchy ═══════════
    // Base exception: maps to Python's RuntimeError
    static auto pyAppError = py::register_exception<AppError>(
        m, "AppError", PyExc_RuntimeError);

    // Derived: maps under AppError
    static auto pyValidation = py::register_exception<ValidationError>(
        m, "ValidationError", pyAppError.ptr());

    static auto pyNotFound = py::register_exception<NotFoundError>(
        m, "NotFoundError", pyAppError.ptr());

    // ═══════════ Custom translator with attributes ═══════════
    py::register_exception_translator([](std::exception_ptr p) {
        try {
            if (p) std::rethrow_exception(p);
        } catch (const NotFoundError& e) {
            py::object exc = pyNotFound(e.what());
            exc.attr("code") = e.code;
            PyErr_SetObject(pyNotFound.ptr(), exc.ptr());
        } catch (const ValidationError& e) {
            py::object exc = pyValidation(e.what());
            exc.attr("field") = e.field;
            PyErr_SetObject(pyValidation.ptr(), exc.ptr());
        }
    });

    m.def("validate_email", &validate_email);
    m.def("find_user", &find_user);
}

```

```python

from exceptions_demo import validate_email, find_user, \
    AppError, ValidationError, NotFoundError

try:
    validate_email("bad-email")
except ValidationError as e:
    print(f"Field: {e.field}, Error: {e}")
    # Field: email, Error: Invalid email: bad-email

try:
    find_user(42)
except NotFoundError as e:
    print(f"Code: {e.code}, Error: {e}")
    # Code: 404, Error: User not found

# Hierarchy works:
try:
    find_user(42)
except AppError:  # catches NotFoundError too
    print("Caught AppError (base)")

```

### Q3: Use py::buffer_protocol to expose a C++ matrix as a NumPy array without copying data

**Answer:**

```cpp

#include <pybind11/pybind11.h>
#include <pybind11/numpy.h>
#include <vector>
#include <stdexcept>
#include <sstream>

namespace py = pybind11;

// ═══════════ C++ Matrix class ═══════════
class Matrix {
    std::vector<double> data_;
    size_t rows_, cols_;

public:
    Matrix(size_t rows, size_t cols)
        : data_(rows * cols, 0.0), rows_(rows), cols_(cols) {}

    double& operator()(size_t r, size_t c) {
        return data_[r * cols_ + c];
    }

    double operator()(size_t r, size_t c) const {
        return data_[r * cols_ + c];
    }

    size_t rows() const { return rows_; }
    size_t cols() const { return cols_; }
    double* data() { return data_.data(); }
    const double* data() const { return data_.data(); }
    size_t size() const { return data_.size(); }

    void fill(double value) {
        std::fill(data_.begin(), data_.end(), value);
    }
};

PYBIND11_MODULE(matrix_mod, m) {
    py::class_<Matrix>(m, "Matrix", py::buffer_protocol())
        .def(py::init<size_t, size_t>())
        .def("fill", &Matrix::fill)
        .def_property_readonly("rows", &Matrix::rows)
        .def_property_readonly("cols", &Matrix::cols)

        // ═══════════ Buffer protocol: expose as NumPy array ═══════════
        .def_buffer([](Matrix& mat) -> py::buffer_info {
            return py::buffer_info(
                mat.data(),                          // Pointer to data
                sizeof(double),                      // Size of one element
                py::format_descriptor<double>::format(), // Python struct format
                2,                                    // Number of dimensions
                {mat.rows(), mat.cols()},            // Shape
                {sizeof(double) * mat.cols(),        // Strides (row-major)
                 sizeof(double)}
            );
        })

        // ═══════════ Construct from NumPy array ═══════════
        .def(py::init([](py::array_t<double> arr) {
            auto buf = arr.unchecked<2>();
            Matrix mat(buf.shape(0), buf.shape(1));
            for (py::ssize_t i = 0; i < buf.shape(0); ++i)
                for (py::ssize_t j = 0; j < buf.shape(1); ++j)
                    mat(i, j) = buf(i, j);
            return mat;
        }))

        .def("__repr__", [](const Matrix& mat) {
            return "<Matrix " + std::to_string(mat.rows()) + "x" +
                   std::to_string(mat.cols()) + ">";
        });
}

```

```python

import numpy as np
from matrix_mod import Matrix

# Create C++ matrix and get NumPy view (zero-copy!)
mat = Matrix(3, 4)
mat.fill(42.0)

# Convert to NumPy array — SHARES memory (no copy)
arr = np.array(mat, copy=False)
print(arr.shape)  # (3, 4)
print(arr[0, 0])  # 42.0

# Modify through NumPy — C++ matrix also changes
arr[1, 2] = 99.0
a2 = np.array(mat, copy=False)
print(a2[1, 2])    # 99.0  ← modified through NumPy

# Construct Matrix from NumPy
np_mat = np.array([[1, 2], [3, 4]], dtype=np.float64)
cpp_mat = Matrix(np_mat)
print(cpp_mat.rows, cpp_mat.cols)  # 2 2

```

---

## Notes

- `py::buffer_protocol()` flag is required in the class definition for `def_buffer` to work
- Buffer protocol gives zero-copy: NumPy and C++ share the same memory
- `PYBIND11_OVERRIDE_PURE` = pure virtual; `PYBIND11_OVERRIDE` = virtual with default
- Trampoline class (PyShape) is needed for any class with virtual methods overridable from Python
- Exception translation follows LIFO order — register most specific first
- `py::register_exception` creates a Python exception class that inherits from the specified base
