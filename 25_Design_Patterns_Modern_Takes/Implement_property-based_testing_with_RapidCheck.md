# Implement property-based testing with RapidCheck

**Category:** Design Patterns — Modern Takes  
**Item:** #755  
**Standard:** C++17  
**Reference:** <https://github.com/emil-e/rapidcheck>  

---

## Topic Overview

**Property-based testing** generates random inputs and verifies that invariants (properties) hold for all of them. Unlike unit tests that check specific examples, property tests explore the input space. **RapidCheck** is the leading C++ framework for this, inspired by Haskell's QuickCheck.

### Unit Tests vs Property Tests

```cpp

Unit test:                           Property test:
  CHECK(sort({3,1,2}) == {1,2,3})     rc::check([] {
  CHECK(sort({}) == {})                   auto v = *rc::gen::arbitrary<vector<int>>();
  CHECK(sort({1}) == {1})                 auto sorted = sort(v);
  // You pick examples                    RC_ASSERT(is_sorted(sorted));
                                          RC_ASSERT(sorted.size() == v.size());
                                       });
                                       // RapidCheck picks hundreds of examples

```

### Core RapidCheck Concepts

| Concept | Purpose |
| --- | --- |
| `rc::check()` | Run a property with random inputs |
| `rc::gen::arbitrary<T>()` | Generate random values of type T |
| `rc::gen::container<C>()` | Generate random containers |
| `rc::gen::inRange(lo, hi)` | Generate bounded integers |
| `RC_ASSERT(cond)` | Assert property holds |
| Shrinking | Auto-minimize failing inputs |

---

## Self-Assessment

### Q1: Write a RapidCheck property that verifies sort(sort(v)) == sort(v) for any input vector

**Answer:**

```cpp

#include <rapidcheck.h>
#include <algorithm>
#include <vector>
#include <iostream>

int main() {
    // ═══════════ Property: sorting is idempotent ═══════════
    rc::check("sort(sort(v)) == sort(v)", [](std::vector<int> v) {
        auto sorted_once = v;
        std::sort(sorted_once.begin(), sorted_once.end());

        auto sorted_twice = sorted_once;
        std::sort(sorted_twice.begin(), sorted_twice.end());

        RC_ASSERT(sorted_once == sorted_twice);
    });

    // ═══════════ Property: sorting preserves elements ═══════════
    rc::check("sort preserves elements", [](std::vector<int> v) {
        auto sorted = v;
        std::sort(sorted.begin(), sorted.end());

        // Same size
        RC_ASSERT(sorted.size() == v.size());

        // Same multiset of elements
        auto v_copy = v;
        std::sort(v_copy.begin(), v_copy.end());
        RC_ASSERT(v_copy == sorted);
    });

    // ═══════════ Property: sorted output is ordered ═══════════
    rc::check("sort produces ordered output", [](std::vector<int> v) {
        std::sort(v.begin(), v.end());
        for (size_t i = 1; i < v.size(); ++i) {
            RC_ASSERT(v[i - 1] <= v[i]);
        }
    });

    std::cout << "All properties passed!\n";
}
// Compile: g++ -std=c++17 -I/path/to/rapidcheck/include -lrapidcheck main.cpp

```

### Q2: Use rc::gen::container to generate arbitrary vectors and rc::gen::inRange for bounded integers

**Answer:**

```cpp

#include <rapidcheck.h>
#include <vector>
#include <numeric>
#include <iostream>

int main() {
    // ═══════════ Bounded integers with inRange ═══════════
    rc::check("ages are valid", []() {
        // Generate age in [0, 150)
        auto age = *rc::gen::inRange(0, 150);
        RC_ASSERT(age >= 0);
        RC_ASSERT(age < 150);
    });

    // ═══════════ Containers with specific element generators ═══════════
    rc::check("vector of positive ints", []() {
        // Generate a vector<int> where every element is in [1, 1000)
        auto v = *rc::gen::container<std::vector<int>>(
            rc::gen::inRange(1, 1000)
        );

        for (auto x : v) {
            RC_ASSERT(x >= 1);
            RC_ASSERT(x < 1000);
        }

        // Sum of positives must be non-negative
        auto sum = std::accumulate(v.begin(), v.end(), 0LL);
        RC_ASSERT(sum >= 0);
    });

    // ═══════════ Fixed-size containers ═══════════
    rc::check("matrix row of exactly 4 elements", []() {
        auto row = *rc::gen::container<std::vector<double>>(
            4, rc::gen::arbitrary<double>()
        );
        RC_ASSERT(row.size() == 4);
    });

    // ═══════════ Combining generators ═══════════
    rc::check("pair of bounded values", []() {
        auto lo = *rc::gen::inRange(0, 50);
        auto hi = *rc::gen::inRange(50, 100);
        RC_ASSERT(lo < hi);  // Always true since ranges don't overlap
    });

    std::cout << "All generator properties passed!\n";
}

```

### Q3: Show that RapidCheck automatically shrinks failing inputs to the minimal reproducing case

**Answer:**

```cpp

#include <rapidcheck.h>
#include <vector>
#include <numeric>
#include <iostream>
#include <algorithm>

// ═══════════ Buggy function: fails for vectors with sum > 100 ═══════════
bool check_sum(const std::vector<int>& v) {
    auto sum = std::accumulate(v.begin(), v.end(), 0);
    return sum <= 100;  // Bug: rejects valid large inputs
}

int main() {
    // RapidCheck will:
    // 1. Generate random vectors (potentially large, e.g. {47, 83, -12, 91, ...})
    // 2. Find one that fails (sum > 100)
    // 3. SHRINK it to the minimal case (e.g. {101} or {51, 50})
    rc::check("sum check bug demo", [](std::vector<int> v) {
        RC_ASSERT(check_sum(v));
    });
    // Output will include something like:
    // Falsifiable after 15 tests and 8 shrinks
    //
    // std::tuple<std::vector<int>>:
    // ({101})              <-- Shrunk to minimal failing case!
    //
    // The original failing input might have been {47, 83, -12, 91, 200, ...}
    // but RapidCheck reduced it to just {101} — the simplest counterexample.

    // ═══════════ Shrinking strings ═══════════
    rc::check("no 'x' in string (intentionally wrong)", [](std::string s) {
        RC_ASSERT(s.find('x') == std::string::npos);
    });
    // Shrinks to: "x"  (minimal string containing 'x')

    // ═══════════ How shrinking works ═══════════
    /*
    RapidCheck's shrinking strategy:
    
    For integers:    tries values closer to 0
                     42 → 21 → 10 → 5 → 3 → 2 → 1 → 0
    
    For containers:  removes elements, then shrinks remaining elements
                     {50, 60, -5} → {50, 60} → {60} → {101} → minimal
    
    For strings:     removes chars, then tries simpler chars
                     "fxyz" → "xyz" → "xz" → "x"
    
    Custom types:    implement rc::Arbitrary<T>::shrink() for domain-specific shrinking
    */
}

```

---

## Notes

- RapidCheck runs **100 test cases by default**; configure with `RC_PARAMS` env variable
- Always check properties that are **universally true** (idempotency, commutativity, round-trip, size preservation)
- Shrinking is RapidCheck's killer feature — it turns "test failed with {23, -7, 89, 0, 44}" into "test failed with {101}"
- Integrate with Google Test via `#include <rapidcheck/gtest.h>` and `RC_GTEST_PROP`
