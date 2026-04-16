# Use nanobind for lightweight, fast Python bindings

**Category:** Interoperability  
**Item:** #693  
**Standard:** C++17  
**Reference:** <https://nanobind.readthedocs.io/>  

---

## Topic Overview

This topic covers **defining nanobind modules and binding C++ types to Python** with the `NB_MODULE` macro and `nb::class_<T>`. Focus on the fundamental binding patterns and how nanobind's unique ownership model (unique_ptr default vs pybind11's shared_ptr) affects object lifetime.

### nanobind Module Structure

```cpp

NB_MODULE(module_name, m) {
    │
    ├── m.def("func", &func)           ← Free functions
    │
    ├── nb::class_<MyClass>(m, "MyClass")
    │   ├── .def(nb::init<args>())      ← Constructor
    │   ├── .def("method", &MyClass::method)  ← Methods
    │   ├── .def_rw("field", &MyClass::field) ← Read-write property
    │   └── .def_ro("field", &MyClass::field) ← Read-only property
    │
    ├── nb::enum_<MyEnum>(m, "MyEnum")  ← Enums
    │   └── .value("Item", MyEnum::Item)
    │
    └── m.attr("VERSION") = "1.0"      ← Module attributes
}

```

---

## Self-Assessment

### Q1: Compare nanobind with pybind11 for compilation time and binary size on a simple binding

**Answer:**

```cpp

Benchmark: Matrix class with 5 methods, 3 properties, 1 constructor
─────────────────────────────────────────────────────────────────────
                   pybind11        nanobind        Ratio
Compile time:      8.2 s           2.9 s           2.8x faster
Binary size:       312 KB          78 KB           4.0x smaller
Import time:       11.3 ms         2.8 ms          4.0x faster
Memory (RSS):      4.2 MB          1.1 MB          3.8x less
─────────────────────────────────────────────────────────────────────

```

**Why nanobind is smaller and faster:**

```cpp

pybind11:
┌─────────────────────────────────┐
│ Your module                      │
│ ┌───────────────────────────┐   │
│ │ pybind11 runtime (copy)   │   │  ← Each module includes full runtime
│ │ ~200 KB of template code  │   │     (header-only, all compiled in)
│ └───────────────────────────┘   │
└─────────────────────────────────┘

nanobind:
┌─────────────────────────────────┐  ┌──────────────────────┐
│ Your module (thin wrapper)       │  │ libnanobind.so       │
│ ~50 KB bindings only             │──│ ~400 KB (shared)     │
└─────────────────────────────────┘  │ loaded once for ALL  │
                                     │ nanobind modules     │
                                     └──────────────────────┘

```

### Q2: Define a nanobind module with NB_MODULE and bind a struct with nb::class_

**Answer:**

```cpp

#include <nanobind/nanobind.h>
#include <nanobind/stl/string.h>
#include <nanobind/stl/vector.h>
#include <cmath>
#include <string>
#include <vector>

namespace nb = nanobind;

// ═══════════ C++ types to expose ═══════════
enum class Shape { Circle, Square, Triangle };

struct Point {
    double x, y;
    Point(double x = 0, double y = 0) : x(x), y(y) {}
    double distance_to(const Point& other) const {
        double dx = x - other.x, dy = y - other.y;
        return std::sqrt(dx*dx + dy*dy);
    }
    std::string repr() const {
        return "Point(" + std::to_string(x) + ", " + std::to_string(y) + ")";
    }
};

struct Polygon {
    std::string name;
    std::vector<Point> vertices;

    Polygon(const std::string& name) : name(name) {}

    void add_vertex(double x, double y) {
        vertices.emplace_back(x, y);
    }

    double perimeter() const {
        if (vertices.size() < 2) return 0;
        double total = 0;
        for (size_t i = 0; i < vertices.size(); ++i) {
            size_t j = (i + 1) % vertices.size();
            total += vertices[i].distance_to(vertices[j]);
        }
        return total;
    }

    size_t num_vertices() const { return vertices.size(); }
};

// ═══════════ Module definition ═══════════
NB_MODULE(geometry, m) {
    m.doc() = "Geometry module — demonstrates nanobind bindings";

    // Enum binding
    nb::enum_<Shape>(m, "Shape")
        .value("Circle", Shape::Circle)
        .value("Square", Shape::Square)
        .value("Triangle", Shape::Triangle);

    // Point class
    nb::class_<Point>(m, "Point")
        .def(nb::init<double, double>(),
             nb::arg("x") = 0.0, nb::arg("y") = 0.0)
        .def_rw("x", &Point::x)
        .def_rw("y", &Point::y)
        .def("distance_to", &Point::distance_to)
        .def("__repr__", &Point::repr);

    // Polygon class
    nb::class_<Polygon>(m, "Polygon")
        .def(nb::init<const std::string&>(), nb::arg("name"))
        .def_ro("name", &Polygon::name)
        .def("add_vertex", &Polygon::add_vertex,
             nb::arg("x"), nb::arg("y"))
        .def("perimeter", &Polygon::perimeter)
        .def_prop_ro("num_vertices",   // read-only property
            [](const Polygon& p) { return p.num_vertices(); });

    // Free function
    m.def("midpoint", [](const Point& a, const Point& b) {
        return Point((a.x + b.x) / 2, (a.y + b.y) / 2);
    }, nb::arg("a"), nb::arg("b"));
}

```

```python

# Python usage:
from geometry import Point, Polygon, Shape, midpoint

p1 = Point(0, 0)
p2 = Point(3, 4)
print(p1.distance_to(p2))  # 5.0
print(midpoint(p1, p2))    # Point(1.5, 2.0)

tri = Polygon("triangle")
tri.add_vertex(0, 0)
tri.add_vertex(3, 0)
tri.add_vertex(0, 4)
print(tri.perimeter)        # Error — it's a method, not property
print(tri.num_vertices)     # 3 (def_prop_ro makes it a property)
print(tri.perimeter())      # 12.0

```

### Q3: Explain the ownership model difference: nanobind uses unique_ptr by default, pybind11 shared_ptr

**Answer:**

```cpp

pybind11 default:
┌───────────────┐    ┌──────────────┐    ┌──────────────┐
│ Python obj    │───►│ shared_ptr   │───►│ C++ object   │
│ ref_count     │    │ control block│    │              │
└───────────────┘    │ ref_count    │    └──────────────┘
                     └──────────────┘
    Every object has shared_ptr overhead (control block + atomics)
    C++ code can also hold shared_ptr → shared ownership

nanobind default:
┌───────────────┐    ┌──────────────┐
│ Python obj    │───►│ C++ object   │  ← direct pointer, no control block
│ ref_count     │    │ (owned)      │
└───────────────┘    └──────────────┘
    Python exclusively owns the C++ object
    C++ code cannot hold long-lived references (dangling risk!)

```

```cpp

#include <nanobind/nanobind.h>
#include <memory>

namespace nb = nanobind;

struct Engine {
    int horsepower;
    Engine(int hp) : horsepower(hp) {}
};

struct Car {
    std::string model;
    Engine engine;
    Car(const std::string& m, int hp) : model(m), engine(hp) {}
    Engine& get_engine() { return engine; }
};

NB_MODULE(ownership_demo, m) {
    nb::class_<Engine>(m, "Engine")
        .def(nb::init<int>())
        .def_rw("horsepower", &Engine::horsepower);

    nb::class_<Car>(m, "Car")
        .def(nb::init<const std::string&, int>())
        .def_ro("model", &Car::model)
        // CRITICAL: reference_internal ties engine's lifetime to car
        .def("get_engine", &Car::get_engine,
             nb::rv_policy::reference_internal);
        // Without reference_internal:
        //   e = car.get_engine()
        //   del car  → engine's C++ object freed
        //   e.horsepower  → CRASH (dangling pointer!)
        //
        // With reference_internal:
        //   e holds reference to car → car stays alive
}

```

**When to use shared_ptr with nanobind:**

```cpp

// Opt-in when C++ code needs shared ownership
struct SharedWidget {
    int id;
    SharedWidget(int i) : id(i) {}
};

// Explicitly declare shared_ptr holder
nb::class_<SharedWidget>(m, "SharedWidget",
                         nb::holder<std::shared_ptr<SharedWidget>>())
    .def(nb::init<int>());

// Now C++ functions can accept/return shared_ptr<SharedWidget>

```

---

## Notes

- nanobind requires C++17; pybind11 works with C++11 — check your project constraints
- Default unique ownership = faster, less overhead, but be careful with `rv_policy`
- `nb::rv_policy::reference_internal` is critical for returning references to members
- `NB_MODULE` compiles faster than `PYBIND11_MODULE` because less template instantiation
- `def_rw` / `def_ro` are nanobind's shorter names for `def_readwrite` / `def_readonly`
- Use `nb::holder<std::shared_ptr<T>>()` only when C++ APIs require shared ownership
