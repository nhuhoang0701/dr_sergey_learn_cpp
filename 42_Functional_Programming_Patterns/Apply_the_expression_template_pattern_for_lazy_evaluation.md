# Apply the expression template pattern for lazy evaluation

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://en.wikipedia.org/wiki/Expression_templates>  

---

## Topic Overview

Expression templates defer computation by building an expression tree at compile time. The actual computation happens only when the result is needed, avoiding temporary objects entirely. This is one of the classic "zero-cost abstraction" techniques in C++, and understanding it will help you see why libraries like Eigen can match hand-written loop performance without sacrificing the readability of math notation.

### The Problem: Temporary Objects in Operator Overloading

The naive approach to operator overloading for math types creates a new object for each operation. That sounds fine for one addition, but chains like `d = a + b + c` become expensive. Here is what that looks like, and why it hurts:

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

Each `+` triggers a full heap allocation and a complete pass over the data. For large vectors with many terms, this is painfully wasteful - you end up doing N passes when one would do.

### Expression Templates Solution

The key insight is: instead of computing `a + b` immediately, return a lightweight object that *describes* the addition. The actual computation gets deferred to the moment you assign the result, at which point you loop just once and evaluate everything element by element.

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

The reason this works is that `a + b` now returns an `AddExpr<ETVec, ETVec>`, not an `ETVec`. Then `(a + b) + c` returns an `AddExpr<AddExpr<ETVec, ETVec>, ETVec>` - still no computation. Only when `operator=` is called does the loop run, calling `expr[i]` which recursively evaluates the whole tree for each element. The compiler inlines all of it and produces a single tight loop, identical to what you'd write by hand.

---

## Self-Assessment

### Q1: How do expression templates achieve zero-overhead abstraction

The compiler inlines all the `operator[]` calls on the expression tree. The resulting code is identical to a hand-written loop: `d[i] = a[i] + b[i] + c[i]`. No temporary vectors are allocated, no extra loops are executed. The expression tree exists only at compile time as a chain of nested template types - at runtime it simply doesn't exist.

### Q2: What libraries use expression templates

Eigen (linear algebra), Blaze (linear algebra), Boost.uBLAS, Armadillo, and xTensor all use expression templates to make `Matrix C = A * B + D` compile to optimized single-pass code. This is a big part of why these libraries can compete with hand-tuned BLAS routines.

### Q3: What are the downsides of expression templates

- Compile times increase significantly because of deep template nesting.
- Error messages are very long and cryptic - when something goes wrong, the type names in the diagnostics are enormous.
- Dangling references become a real danger if expressions accidentally capture temporaries by reference instead of by value.
- Debugging is harder because the expression tree is invisible in the debugger.
- Code complexity for library authors is high - this is expert-level template metaprogramming.

---

## Notes

- Expression templates are the classic C++ "zero-cost abstraction" technique for math-heavy code.
- C++20 concepts can improve error messages for expression template libraries by constraining what types are accepted.
- The technique predates C++11 - it was invented by Todd Veldhuizen in 1995.
- Modern alternatives worth knowing: ranges (lazy pipelines for sequences) and `std::mdspan` (for multidimensional array access patterns).
