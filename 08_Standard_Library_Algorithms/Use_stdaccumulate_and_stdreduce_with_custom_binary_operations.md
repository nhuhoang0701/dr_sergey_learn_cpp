# Use std::accumulate and std::reduce with custom binary operations

**Category:** Standard Library — Algorithms  
**Item:** #73  
**Standard:** C++11 / C++17  
**Reference:** <https://en.cppreference.com/w/cpp/algorithm/accumulate>  

---

## Topic Overview

### std::accumulate (C++03, `<numeric>`)

Left-folds a range with a binary operation, strictly left-to-right, sequentially:

```cpp

accumulate(first, last, init, op) =
    op(op(op(init, *first), *(first+1)), *(first+2))...

```

Default operation is `+`. Result type is the type of `init` (beware: `accumulate(v.begin(), v.end(), 0)` on a vector of doubles truncates to `int`!).

### std::reduce (C++17, `<numeric>`)

Similar to `accumulate` but the operation execution order is **unspecified** — it can be parallelized:

```cpp

reduce(first, last, init, op) =
    op applied in any order/grouping (associative+commutative required)

```

| Feature | `accumulate` | `reduce` |
| --- | --- | --- |
| Order | Strict left-to-right | Unspecified |
| Parallelism | ❌ Not parallelizable | ✅ Supports execution policies |
| Op requirement | None | Associative + commutative |
| Default init | Must provide | 0 (value-initialized) optional |
| Header | `<numeric>` | `<numeric>` |

### std::inner_product (C++03, `<numeric>`)

Computes `init + a[0]*b[0] + a[1]*b[1] + ...` with customizable `+` and `*` operations.

---

## Self-Assessment

### Q1: Compute the product of all elements in a vector using std::accumulate with a lambda

```cpp

#include <iostream>
#include <vector>
#include <numeric>
#include <string>

int main() {
    std::vector<int> v = {2, 3, 4, 5};

    // === Product using accumulate with lambda ===
    int product = std::accumulate(v.begin(), v.end(), 1,
                                  [](int acc, int x) { return acc * x; });
    std::cout << "Product: " << product << "\n";  // 120 (2*3*4*5)

    // === Sum (default operation) ===
    int sum = std::accumulate(v.begin(), v.end(), 0);
    std::cout << "Sum: " << sum << "\n";  // 14

    // === String concatenation ===
    std::vector<std::string> words = {"Hello", " ", "World", "!"};
    std::string sentence = std::accumulate(
        words.begin(), words.end(), std::string{},
        [](std::string acc, const std::string& w) { return acc + w; }
    );
    std::cout << "Sentence: " << sentence << "\n";  // Hello World!

    // Note: string concatenation with accumulate is O(n²) due to repeated copying.
    // For large strings, prefer a different approach.

    // === Finding maximum via accumulate ===
    std::vector<int> data = {7, 2, 9, 4, 1, 8};
    int max_val = std::accumulate(data.begin(), data.end(), data.front(),
                                  [](int a, int b) { return a > b ? a : b; });
    std::cout << "Max: " << max_val << "\n";  // 9

    // === BEWARE: init type determines result type! ===
    std::vector<double> dv = {1.5, 2.5, 3.5};
    int wrong = std::accumulate(dv.begin(), dv.end(), 0);       // 0 is int!
    double right = std::accumulate(dv.begin(), dv.end(), 0.0);  // 0.0 is double
    std::cout << "Wrong (int init): " << wrong << "\n";   // 6 (truncated!)
    std::cout << "Right (double init): " << right << "\n"; // 7.5

    return 0;
}

```

**How it works:**

- `std::accumulate` takes `(first, last, init, binary_op)`.
- `binary_op(accumulator, current_element)` is applied left-to-right.
- The init value must be the identity element for the operation (1 for multiplication, 0 for addition, `""` for concatenation).
- The result type matches the type of `init` — this is the most common source of bugs.

### Q2: Explain why std::reduce can be parallelized but std::accumulate cannot

`std::accumulate` guarantees **strict left-to-right** application:

```cpp

result = op(op(op(init, a[0]), a[1]), a[2])

```

This is inherently sequential — you must know the result of `op(init, a[0])` before computing `op(result, a[1])`.

`std::reduce` allows **any grouping** and **any order**:

```cpp

Possible: op(op(init, a[0]), op(a[1], a[2]))     ← tree reduction
Possible: op(op(a[2], a[0]), op(init, a[1]))     ← reordered

```

This freedom enables parallelism — split the range across threads, reduce independently, merge results.

```cpp

#include <iostream>
#include <vector>
#include <numeric>
#include <execution>
#include <chrono>

int main() {
    std::vector<double> v(10'000'000, 1.0);

    // === Sequential accumulate ===
    auto t1 = std::chrono::high_resolution_clock::now();
    double sum_acc = std::accumulate(v.begin(), v.end(), 0.0);
    auto t2 = std::chrono::high_resolution_clock::now();
    auto ms_acc = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === Sequential reduce (same speed, but allows parallelism) ===
    t1 = std::chrono::high_resolution_clock::now();
    double sum_seq = std::reduce(std::execution::seq, v.begin(), v.end(), 0.0);
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_seq = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // === Parallel reduce ===
    t1 = std::chrono::high_resolution_clock::now();
    double sum_par = std::reduce(std::execution::par, v.begin(), v.end(), 0.0);
    t2 = std::chrono::high_resolution_clock::now();
    auto ms_par = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    std::cout << "accumulate:     " << sum_acc << " in " << ms_acc << " ms\n";
    std::cout << "reduce (seq):   " << sum_seq << " in " << ms_seq << " ms\n";
    std::cout << "reduce (par):   " << sum_par << " in " << ms_par << " ms\n";

    // === Why reduce requires associativity + commutativity ===
    // Subtraction is NOT associative:
    //   (10 - 3) - 2 = 5  ≠  10 - (3 - 2) = 9
    // Using it with reduce gives WRONG results:
    std::vector<int> nums = {10, 3, 2};
    int sub_acc = std::accumulate(nums.begin(), nums.end(), 0,
                                  std::minus<>{});
    // accumulate: 0 - 10 - 3 - 2 = -15 (well-defined)

    int sub_red = std::reduce(nums.begin(), nums.end(), 0,
                              std::minus<>{});
    // reduce: order unspecified — could be anything!
    // This is a CONTRACT VIOLATION (UB in practice)

    std::cout << "accumulate with minus: " << sub_acc << "\n";
    std::cout << "reduce with minus:     " << sub_red << " (unreliable!)\n";

    return 0;
}

```

### Q3: Use std::inner_product to compute a weighted sum

```cpp

#include <iostream>
#include <vector>
#include <numeric>

int main() {
    // === Weighted sum: w[0]*v[0] + w[1]*v[1] + ... ===
    std::vector<double> values  = {90.0, 85.0, 78.0, 95.0};
    std::vector<double> weights = {0.3,  0.2,  0.2,  0.3};

    double weighted_sum = std::inner_product(
        values.begin(), values.end(),
        weights.begin(),
        0.0  // init
    );
    // Default ops: + for accumulation, * for combining pairs
    // = 0.0 + 90*0.3 + 85*0.2 + 78*0.2 + 95*0.3
    // = 0.0 + 27.0 + 17.0 + 15.6 + 28.5 = 88.1
    std::cout << "Weighted average: " << weighted_sum << "\n";  // 88.1

    // === Dot product (same thing, default ops) ===
    std::vector<double> a = {1.0, 2.0, 3.0};
    std::vector<double> b = {4.0, 5.0, 6.0};
    double dot = std::inner_product(a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "Dot product: " << dot << "\n";  // 32 (1*4 + 2*5 + 3*6)

    // === Custom ops: count mismatches between two strings ===
    std::string s1 = "abcdef";
    std::string s2 = "abxdyf";
    int mismatches = std::inner_product(
        s1.begin(), s1.end(), s2.begin(),
        0,
        std::plus<>{},  // accumulate with +
        [](char a, char b) { return a != b ? 1 : 0; }  // pair op
    );
    std::cout << "Mismatches: " << mismatches << "\n";  // 2 (x≠c, y≠e)

    // === Manhattan distance with custom ops ===
    std::vector<int> p1 = {1, 3, 5};
    std::vector<int> p2 = {4, 1, 8};
    int manhattan = std::inner_product(
        p1.begin(), p1.end(), p2.begin(),
        0,
        std::plus<>{},  // sum of ...
        [](int a, int b) { return std::abs(a - b); }  // absolute differences
    );
    std::cout << "Manhattan distance: " << manhattan << "\n";  // |1-4|+|3-1|+|5-8| = 8

    // === C++17 alternative: std::transform_reduce ===
    // inner_product is to accumulate as transform_reduce is to reduce
    double dot2 = std::transform_reduce(a.begin(), a.end(), b.begin(), 0.0);
    std::cout << "transform_reduce dot: " << dot2 << "\n";  // 32

    return 0;
}

```

**How it works:**

- `inner_product(first1, last1, first2, init, op1, op2)` applies `op2` to each pair of elements, then accumulates with `op1`.
- Default `op1 = +`, `op2 = *` gives the standard dot product / weighted sum.
- The C++17 `std::transform_reduce` is the parallelizable equivalent.
- Ensure the second range has at least as many elements as the first — no bounds checking!

---

## Notes

- **`accumulate` init type trap:** `accumulate(v.begin(), v.end(), 0)` on `vector<double>` truncates every addition to `int`. Always match init type to desired result type.
- **`reduce` without init:** `reduce(first, last)` uses `T{}` as init (value-initialized). For `int`, that's 0. For custom types, ensure default construction is the identity.
- **`transform_reduce`** is the parallel-friendly replacement for `inner_product`. Prefer it in new C++17+ code.
- **`inclusive_scan` / `exclusive_scan`** are the parallel versions of `partial_sum` — related but different use case.
- **Floating-point reduce:** Because `reduce` may group operations differently, floating-point results may differ slightly from `accumulate` due to different rounding. This is usually acceptable.

## Notes

_Add your own notes, examples, and observations here._

```cpp

// Your practice code

```
