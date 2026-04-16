# Apply the expression template pattern for lazy evaluation

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Expression_templates>  

---

## Topic Overview

Expression templates defer computation by building an expression tree at compile time. The actual computation happens only when the result is needed, avoiding temporary objects.

### The Problem: Temporary Objects in Operator Overloading

```cpp

#include <vector>
#include <cstddef>

class Vec {
    std::vector<double> data_;
public:
    Vec(size_t n) : data_(n) {}
    double& operator[](size_t i) { return data_[i]; }
    double operator[](size_t i) const { return data_[i]; }
    size_t size() const { return data_.size(); }
};

// Naive operator+: creates a temporary Vec
Vec operator+(const Vec& a, const Vec& b) {
    Vec result(a.size());
    for (size_t i = 0; i < a.size(); ++i)
        result[i] = a[i] + b[i];
    return result;
}

// d = a + b + c creates TWO temporaries:
// temp1 = a + b   (allocates, computes)
// temp2 = temp1 + c (allocates, computes)
// d = temp2

```

### Expression Templates Solution

```cpp

#include <cstddef>
#include <vector>
#include <iostream>

// Expression node: addition of two expressions
template<typename L, typename R>
struct AddExpr {
    const L& left;
    const R& right;
    double operator[](size_t i) const { return left[i] + right[i]; }
    size_t size() const { return left.size(); }
};

class ETVec {
    std::vector<double> data_;
public:
    ETVec(size_t n, double val = 0) : data_(n, val) {}

    // Assign from any expression — this is where computation happens
    template<typename Expr>
    ETVec& operator=(const Expr& expr) {
        for (size_t i = 0; i < data_.size(); ++i)
            data_[i] = expr[i];  // Evaluates lazily, element by element
        return *this;
    }

    double operator[](size_t i) const { return data_[i]; }
    size_t size() const { return data_.size(); }
};

// operator+ returns an expression, NOT a computed Vec
template<typename L, typename R>
AddExpr<L, R> operator+(const L& a, const R& b) {
    return {a, b};
}

int main() {
    ETVec a(1000, 1.0), b(1000, 2.0), c(1000, 3.0);
    ETVec d(1000);

    // d = a + b + c
    // Compiled to: d[i] = a[i] + b[i] + c[i] — ONE loop, ZERO temporaries!
    d = a + b + c;

    std::cout << d[0] << "\n"; // 6.0
}

```

---

## Self-Assessment

### Q1: How do expression templates achieve zero-overhead abstraction

The compiler inlines all the `operator[]` calls on the expression tree. The resulting code is identical to a hand-written loop: `d[i] = a[i] + b[i] + c[i]`. No temporary vectors are allocated, no extra loops are executed. The expression tree exists only at compile time.

### Q2: What libraries use expression templates

Eigen (linear algebra), Blaze (linear algebra), Boost.uBLAS, Armadillo, and xTensor. They all use expression templates to make `Matrix C = A * B + D` compile to optimized single-pass code.

### Q3: What are the downsides of expression templates

- Compile times increase significantly (deep template nesting).
- Error messages are very long and cryptic.
- Dangling references if expressions capture temporaries.
- Debugging is harder (expression tree not visible in debugger).
- Code complexity for library authors is high.

---

## Notes

- Expression templates are the classic C++ "zero-cost abstraction" technique.
- C++20 concepts can improve error messages for expression template libraries.
- The technique predates C++11 — invented by Todd Veldhuizen in 1995.
- Modern alternatives: ranges (lazy pipelines) and `std::mdspan` (for multidimensional arrays).
