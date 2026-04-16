# Use the Strategy pattern with templates for zero-overhead policy injection

**Category:** Design Patterns — Modern Takes  
**Item:** #673  
**Reference:** <https://en.wikipedia.org/wiki/Strategy_pattern>  

---

## Topic Overview

**Policy-based design** uses template parameters to inject behavior at compile time. Unlike `std::function` (which erases types and adds indirection), template policies let the compiler **inline** the strategy entirely, producing code as efficient as hand-written specializations — a "pay for what you use" approach.

### Multi-Policy Design

```cpp

Sorter<ComparePolicy, PartitionPolicy>
    │           │              │
    │    AscendingCmp     LomutoPartition     → Sorter<AscendingCmp, LomutoPartition>
    │    DescendingCmp    HoarePartition      → Sorter<DescendingCmp, HoarePartition>
    │    CustomCmp        MedianOf3Partition   → Sorter<CustomCmp, MedianOf3Partition>
    │
    └── Each combination is a unique type with fully inlined behavior

```

---

## Self-Assessment

### Q1: Implement a Sorter<ComparePolicy, PartitionPolicy> that composes sorting behavior at compile time

**Answer:**

```cpp

#include <iostream>
#include <vector>
#include <algorithm>
#include <utility>

// ═══════════ Compare Policies ═══════════
struct AscendingCmp {
    template<typename T>
    bool operator()(const T& a, const T& b) const { return a < b; }
};

struct DescendingCmp {
    template<typename T>
    bool operator()(const T& a, const T& b) const { return a > b; }
};

// ═══════════ Partition Policies ═══════════
struct LomutoPartition {
    template<typename Iter, typename Cmp>
    Iter operator()(Iter lo, Iter hi, Cmp cmp) const {
        auto pivot = *std::prev(hi);
        auto i = lo;
        for (auto j = lo; j != std::prev(hi); ++j) {
            if (cmp(*j, pivot)) {
                std::iter_swap(i, j);
                ++i;
            }
        }
        std::iter_swap(i, std::prev(hi));
        return i;
    }
};

struct HoarePartition {
    template<typename Iter, typename Cmp>
    Iter operator()(Iter lo, Iter hi, Cmp cmp) const {
        auto pivot = *lo;
        auto i = lo;
        auto j = std::prev(hi);
        while (true) {
            while (cmp(*i, pivot)) ++i;
            while (cmp(pivot, *j)) --j;
            if (i >= j) return j;
            std::iter_swap(i, j);
            ++i; --j;
        }
    }
};

// ═══════════ Sorter: composed from two policies ═══════════
template<typename ComparePolicy = AscendingCmp,
         typename PartitionPolicy = LomutoPartition>
class Sorter {
    ComparePolicy cmp_;
    PartitionPolicy partition_;

    template<typename Iter>
    void quicksort(Iter lo, Iter hi) {
        if (lo >= hi || std::next(lo) == hi) return;
        auto pivot = partition_(lo, hi, cmp_);
        quicksort(lo, pivot);
        quicksort(std::next(pivot), hi);
    }

public:
    template<typename Container>
    void sort(Container& c) {
        quicksort(c.begin(), c.end());
    }
};

int main() {
    std::vector<int> data1 = {5, 2, 8, 1, 9, 3};
    Sorter<AscendingCmp, LomutoPartition> asc_sorter;
    asc_sorter.sort(data1);
    for (int x : data1) std::cout << x << ' ';  // 1 2 3 5 8 9
    std::cout << '\n';

    std::vector<int> data2 = {5, 2, 8, 1, 9, 3};
    Sorter<DescendingCmp, LomutoPartition> desc_sorter;
    desc_sorter.sort(data2);
    for (int x : data2) std::cout << x << ' ';  // 9 8 5 3 2 1
    std::cout << '\n';
}

```

### Q2: Show that the template strategy generates inlined code with no virtual dispatch overhead

**Answer:**

```cpp

#include <iostream>
#include <chrono>
#include <vector>
#include <algorithm>
#include <functional>

// ═══════════ Template policy: compiles to inlined code ═══════════
struct InlinedCompare {
    bool operator()(int a, int b) const { return a < b; }
};

template<typename Cmp>
void template_sort(std::vector<int>& v, Cmp cmp) {
    std::sort(v.begin(), v.end(), cmp);
    // Compiler sees the exact type of cmp → inlines operator()
    // No function pointer, no vtable lookup
}

// ═══════════ Virtual dispatch: runtime indirection ═══════════
struct ICompare {
    virtual ~ICompare() = default;
    virtual bool compare(int a, int b) const = 0;
};

struct VirtualAscending : ICompare {
    bool compare(int a, int b) const override { return a < b; }
};

void virtual_sort(std::vector<int>& v, const ICompare& cmp) {
    std::sort(v.begin(), v.end(), [&](int a, int b) {
        return cmp.compare(a, b);  // Virtual call per comparison!
    });
}

int main() {
    constexpr int N = 5'000'000;

    // Benchmark template policy
    std::vector<int> v1(N);
    std::iota(v1.begin(), v1.end(), 0);
    std::reverse(v1.begin(), v1.end());
    auto t1 = std::chrono::high_resolution_clock::now();
    template_sort(v1, InlinedCompare{});
    auto t2 = std::chrono::high_resolution_clock::now();

    // Benchmark virtual dispatch
    std::vector<int> v2(N);
    std::iota(v2.begin(), v2.end(), 0);
    std::reverse(v2.begin(), v2.end());
    VirtualAscending vcmp;
    auto t3 = std::chrono::high_resolution_clock::now();
    virtual_sort(v2, vcmp);
    auto t4 = std::chrono::high_resolution_clock::now();

    using ms = std::chrono::duration<double, std::milli>;
    std::cout << "Template policy: " << ms(t2 - t1).count() << " ms\n";
    std::cout << "Virtual dispatch: " << ms(t4 - t3).count() << " ms\n";
    // Template policy ≈ same as raw std::sort (inlined)
    // Virtual dispatch ~20-50% slower (indirect call prevents inlining)

    /*
    Why template is faster:

    - Compiler knows exact type → inlines compare function
    - No branch prediction penalty from indirect calls
    - Enables SIMD auto-vectorization in tight loops
    - No vtable pointer dereference

    */
}

```

### Q3: Compare template strategy with std::function strategy for stored vs inline callables

**Answer:**

```cpp

#include <iostream>
#include <functional>
#include <vector>
#include <string>
#include <algorithm>

// ═══════════ Template policy: inline, compile-time fixed ═══════════
template<typename ValidationPolicy>
class TemplateValidator {
    ValidationPolicy policy_;
public:
    bool validate(const std::string& input) {
        return policy_(input);  // Inlined — compiler sees exact function body
    }
};

struct NotEmpty {
    bool operator()(const std::string& s) const { return !s.empty(); }
};
struct MaxLength {
    size_t max;
    bool operator()(const std::string& s) const { return s.size() <= max; }
};

// ═══════════ std::function: type-erased, runtime swappable ═══════════
class RuntimeValidator {
    std::function<bool(const std::string&)> policy_;
public:
    explicit RuntimeValidator(std::function<bool(const std::string&)> p)
        : policy_(std::move(p)) {}

    void set_policy(std::function<bool(const std::string&)> p) {
        policy_ = std::move(p);  // Can change at runtime!
    }

    bool validate(const std::string& input) {
        return policy_(input);  // Indirect call — not inlined
    }
};

int main() {
    // Template: strategy fixed at compile time
    TemplateValidator<NotEmpty> v1;
    TemplateValidator<MaxLength> v2{MaxLength{10}};

    std::cout << v1.validate("hello") << '\n';       // 1
    std::cout << v2.validate("hello world!") << '\n'; // 0 (12 > 10)

    // std::function: strategy changeable at runtime
    RuntimeValidator rv([](const std::string& s) { return !s.empty(); });
    std::cout << rv.validate("hello") << '\n';  // 1

    // Swap strategy based on user config (not possible with templates!)
    bool strict_mode = true;
    if (strict_mode) {
        rv.set_policy([](const std::string& s) {
            return !s.empty() && s.size() <= 10 && s.find(' ') == std::string::npos;
        });
    }
    std::cout << rv.validate("hello world") << '\n';  // 0 (has space)
    std::cout << rv.validate("hello") << '\n';         // 1

    /*
    Decision guide:
    ┌──────────────────────────────┬─────────────────┬──────────────────┐
    │ Need                         │ Use Template    │ Use std::function│
    ├──────────────────────────────┼─────────────────┼──────────────────┤
    │ Maximum performance          │ ✓               │                  │
    │ Strategy known at compile    │ ✓               │                  │
    │ Swap strategy at runtime     │                 │ ✓                │
    │ Plugin / user-configured     │                 │ ✓                │
    │ Minimize binary size         │                 │ ✓                │
    │ Strategy captures state      │ ✓ (functor)     │ ✓ (closure)     │
    └──────────────────────────────┴─────────────────┴──────────────────┘
    */
}

```

---

## Notes

- Template policies are the backbone of STL design: `std::sort` takes a comparator policy, `std::allocator` is a policy
- Multi-policy classes (2+ template params) should provide sensible defaults: `Sorter<> s;` uses default policies
- `std::function` costs ~40 bytes of storage + type erasure; template policies add 0 bytes per instance
- Combine both: use template policy for the hot path, `std::function` for configuration/callbacks that run rarely

// Your practice code

```text
