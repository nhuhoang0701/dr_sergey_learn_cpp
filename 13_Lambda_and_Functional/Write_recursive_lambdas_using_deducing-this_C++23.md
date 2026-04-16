# Write recursive lambdas using deducing-this (C++23)

**Category:** Lambda & Functional  
**Item:** #168  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/language/lambda>  

---

## Topic Overview

Before C++23, recursive lambdas required either `std::function` (with overhead) or passing the lambda to itself as an argument (awkward). C++23's **deducing-this** (`this auto& self`) gives lambdas a direct way to refer to themselves.

```cpp

// C++23: deducing-this — clean and zero-overhead!
auto fib = [](this auto& self, int n) -> int {
    return n <= 1 ? n : self(n - 1) + self(n - 2);
};

fib(10);  // 55

```

### Comparison

| Approach | Overhead | Syntax | Standard |
| --- | --- | --- | --- |
| `std::function` | Heap + virtual dispatch | `std::function<int(int)> fib = [&fib]...` | C++11 |
| Pass-self trick | Zero | `auto fib = [](auto& self, int n) { self(self, n-1); }` | C++14 |
| Y-combinator | Zero | Complex | C++14 |
| **Deducing-this** | **Zero** | `[](this auto& self, int n) { self(n-1); }` | **C++23** |

---

## Self-Assessment

### Q1: Write a recursive Fibonacci lambda using deducing-this without `std::function`

**Solution:**

```cpp

#include <iostream>

int main() {
    // C++23: explicit object parameter (deducing-this)
    auto fib = [](this auto& self, int n) -> int {
        if (n <= 1) return n;
        return self(n - 1) + self(n - 2);
    };

    for (int i = 0; i <= 10; ++i) {
        std::cout << "fib(" << i << ") = " << fib(i) << "\n";
    }

    // Recursive factorial:
    auto factorial = [](this auto& self, int n) -> long long {
        return n <= 1 ? 1LL : n * self(n - 1);
    };

    std::cout << "\n12! = " << factorial(12) << "\n";

    // Recursive tree walk:
    struct Node {
        int value;
        Node* left = nullptr;
        Node* right = nullptr;
    };

    Node n4{4}, n5{5}, n2{2, &n4, &n5}, n3{3}, n1{1, &n2, &n3};

    auto inorder = [](this auto& self, const Node* node) -> void {
        if (!node) return;
        self(node->left);
        std::cout << node->value << " ";
        self(node->right);
    };

    std::cout << "Inorder: ";
    inorder(&n1);
    std::cout << "\n";
}
// Expected output:
//   fib(0) = 0
//   fib(1) = 1
//   fib(2) = 1
//   fib(3) = 2
//   fib(4) = 3
//   fib(5) = 5
//   fib(6) = 8
//   fib(7) = 13
//   fib(8) = 21
//   fib(9) = 34
//   fib(10) = 55
//
//   12! = 479001600
//   Inorder: 4 2 5 1 3

```

---

### Q2: Explain why `std::function` for recursion adds overhead compared to deducing-this

**Solution:**

```cpp

#include <iostream>
#include <functional>
#include <chrono>

int main() {
    // ─── Approach 1: std::function (C++11) ───
    std::function<long long(int)> fib_func;
    fib_func = [&fib_func](int n) -> long long {
        return n <= 1 ? n : fib_func(n - 1) + fib_func(n - 2);
    };

    // ─── Approach 2: deducing-this (C++23) ───
    auto fib_self = [](this auto& self, int n) -> long long {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    };

    // Both produce same result:
    std::cout << "std::function fib(30): " << fib_func(30) << "\n";
    std::cout << "deducing-this fib(30): " << fib_self(30) << "\n";

    // Benchmark:
    auto bench = [](auto fn, int n, const char* name) {
        auto start = std::chrono::steady_clock::now();
        auto result = fn(n);
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(
            std::chrono::steady_clock::now() - start).count();
        std::cout << name << ": " << result << " (" << us << " us)\n";
    };

    bench(fib_func, 35, "std::function");
    bench(fib_self, 35, "deducing-this");

    // WHY std::function is slower:
    std::cout << "\nOverhead analysis:\n";
    std::cout << "  sizeof(fib_func): " << sizeof(fib_func) << " bytes (heap alloc)\n";
    std::cout << "  sizeof(fib_self): " << sizeof(fib_self) << " bytes (stack only)\n";
    std::cout << "\n";
    std::cout << "  std::function overhead per call:\n";
    std::cout << "    1. Type-erased virtual dispatch (indirect call)\n";
    std::cout << "    2. Cannot be inlined by optimizer\n";
    std::cout << "    3. Possible heap allocation for captures\n";
    std::cout << "\n";
    std::cout << "  deducing-this overhead per call:\n";
    std::cout << "    - NONE. Direct call, fully inlinable.\n";
}
// Expected output:
//   std::function fib(30): 832040
//   deducing-this fib(30): 832040
//   std::function: 9227465 (~50000 us)   ← SLOW
//   deducing-this: 9227465 (~20000 us)   ← ~2-3x FASTER
//   sizeof(fib_func): 32 bytes (heap alloc)
//   sizeof(fib_self): 1 bytes (stack only)

```

---

### Q3: Show the Y-combinator for recursive lambdas in C++14 and compare with C++23

**Solution:**

```cpp

#include <iostream>

// ─── C++14: Y-combinator ───
// A fixed-point combinator that enables recursion without naming:
template <typename F>
struct YCombinator {
    F func;

    template <typename... Args>
    auto operator()(Args&&... args) const {
        return func(*this, std::forward<Args>(args)...);
    }
};

// Deduction guide (C++17) or make_y helper (C++14)
template <typename F>
YCombinator<std::decay_t<F>> make_y(F&& f) {
    return {std::forward<F>(f)};
}

int main() {
    // ─── C++14: pass-self trick ───
    auto fib_14 = [](auto& self, int n) -> int {
        return n <= 1 ? n : self(self, n - 1) + self(self, n - 2);
    };                                  // ^^^^ must pass self twice!
    // Must call: fib_14(fib_14, 10)
    std::cout << "C++14 self-pass: fib(10) = " << fib_14(fib_14, 10) << "\n";

    // ─── C++14: Y-combinator wraps the self-passing ───
    auto fib_y = make_y([](auto& self, int n) -> int {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    });                              // ^^^^ clean call syntax
    // Clean call: fib_y(10)
    std::cout << "C++14 Y-comb:    fib(10) = " << fib_y(10) << "\n";

    // ─── C++23: deducing-this (cleanest!) ───
    auto fib_23 = [](this auto& self, int n) -> int {
        return n <= 1 ? n : self(n - 1) + self(n - 2);
    };
    // Clean call: fib_23(10)
    std::cout << "C++23 deducing:  fib(10) = " << fib_23(10) << "\n";

    // ─── Comparison ───
    std::cout << "\n";
    std::cout << "Approach         | Syntax at call-site | Overhead\n";
    std::cout << "---------------- | ------------------- | --------\n";
    std::cout << "self-pass (C++14)| fib(fib, 10)        | Zero\n";
    std::cout << "Y-combinator     | fib(10)             | Zero\n";
    std::cout << "std::function    | fib(10)             | HIGH\n";
    std::cout << "deducing-this    | fib(10)             | Zero\n";
}
// Expected output:
//   C++14 self-pass: fib(10) = 55
//   C++14 Y-comb:    fib(10) = 55
//   C++23 deducing:  fib(10) = 55
// ...

```

---

## Notes

- **`this auto& self`** is the explicit object parameter syntax. The `this` keyword here denotes the first parameter receives the object itself.
- **Mutable recursive lambda:** Use `this auto self` (by value) if you need mutation. Use `this auto& self` (by reference) for the common case.
- **No `std::function` needed:** Deducing-this completely eliminates the need for `std::function` in recursive lambdas.
- **Compiler support:** GCC 14+, Clang 18+, MSVC 19.37+ support deducing-this.
- **Performance:** In benchmarks, deducing-this recursive lambdas match hand-written recursive functions exactly.
