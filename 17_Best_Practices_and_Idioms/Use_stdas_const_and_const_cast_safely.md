# Use std::as_const and const_cast safely

**Category:** Best Practices & Idioms  
**Item:** #136  
**Reference:** <https://en.cppreference.com/w/cpp/utility/as_const>  

---

## Topic Overview

`std::as_const(x)` returns a `const` reference to `x`, letting you force a const overload without modifying the variable. `const_cast` removes const — it's safe only when the original object was non-const.

```cpp

std::string s = "hello";
const std::string& cs = std::as_const(s);  // safe const view

// const_cast: safe ONLY if original was non-const
std::string& back = const_cast<std::string&>(cs);  // OK: s is non-const

```

---

## Self-Assessment

### Q1: Use `std::as_const` to call a const overload

```cpp

#include <iostream>
#include <string>
#include <utility>  // std::as_const
#include <vector>

class Container {
    std::vector<int> data_ = {10, 20, 30};
public:
    // Non-const: returns mutable reference
    std::vector<int>& items() {
        std::cout << "[mutable overload]\n";
        return data_;
    }

    // Const: returns const reference
    const std::vector<int>& items() const {
        std::cout << "[const overload]\n";
        return data_;
    }
};

int main() {
    Container c;

    // Without as_const: calls non-const overload
    auto& v1 = c.items();         // mutable overload

    // With as_const: forces const overload
    auto& v2 = std::as_const(c).items();  // const overload

    // Useful in range-for to avoid copying:
    for (const auto& item : std::as_const(c).items()) {
        std::cout << item << ' ';
    }
    std::cout << '\n';
}
// Expected output:
// [mutable overload]
// [const overload]
// [const overload]
// 10 20 30

```

### Q2: When is `const_cast` safe vs UB

```cpp

#include <iostream>
#include <cstring>

// SAFE: casting away const on a non-const original
void safe_example() {
    int x = 42;                       // x is NON-const
    const int& cref = x;              // const reference to non-const object
    int& ref = const_cast<int&>(cref); // SAFE: x was never const
    ref = 100;                         // OK: modifying non-const object
    std::cout << "Safe: " << x << '\n';
}

// UNDEFINED BEHAVIOR: casting away const on a truly const object
void unsafe_example() {
    const int x = 42;                  // x IS const
    int& ref = const_cast<int&>(x);    // technically compiles...
    // ref = 100;                       // UB! x lives in read-only memory
    // Compiler may have inlined x=42 everywhere
    std::cout << "UB avoided (didn't modify)\n";
}

// COMMON LEGITIMATE USE: calling non-const API with const data
void legacy_api(char* buf) {  // old C API, doesn't modify buf
    std::cout << "Legacy: " << buf << '\n';
}

void call_legacy() {
    const char* msg = "hello";
    // Safe IF we know legacy_api doesn't modify the buffer
    legacy_api(const_cast<char*>(msg));
}

int main() {
    safe_example();
    unsafe_example();
    call_legacy();
}
// Expected output:
// Safe: 100
// UB avoided (didn't modify)
// Legacy: hello

```

| Scenario | Safe? |
| --- | --- |
| Cast away const, original was non-const | Yes |
| Cast away const, original was const | **UB** if modified |
| Cast away const to call legacy C API | Safe if API doesn't modify |
| Adding const (const_cast<const T&>) | Always safe |

### Q3: Show a design flaw where `const_cast` hints at a needed refactor

```cpp

#include <iostream>
#include <string>
#include <unordered_map>

// BAD DESIGN: needs const_cast because interface is wrong
class BadCache {
    mutable std::unordered_map<std::string, int> cache_;  // should be mutable!
    int compute(const std::string& key) const { return key.length() * 42; }
public:
    // Returns cached value, but lookup modifies cache... oops
    int get(const std::string& key) const {
        // Developer uses const_cast because get() is const but cache_ needs mutation
        auto* self = const_cast<BadCache*>(this);
        if (auto it = self->cache_.find(key); it != self->cache_.end())
            return it->second;
        int val = compute(key);
        self->cache_[key] = val;  // modifying through const_cast!
        return val;
    }
};

// GOOD DESIGN: use mutable keyword instead
class GoodCache {
    mutable std::unordered_map<std::string, int> cache_;  // mutable = logical const
    int compute(const std::string& key) const { return key.length() * 42; }
public:
    int get(const std::string& key) const {
        if (auto it = cache_.find(key); it != cache_.end())
            return it->second;
        int val = compute(key);
        cache_[key] = val;  // OK: mutable member
        return val;
    }
};

int main() {
    GoodCache cache;
    std::cout << cache.get("hello") << '\n';  // 210
    std::cout << cache.get("hello") << '\n';  // 210 (cached)
    std::cout << cache.get("world") << '\n';  // 210
}
// Expected output:
// 210
// 210
// 210

```

**When you see `const_cast`, ask:**

1. Should the member be `mutable`? (caches, mutexes, debug counters)
2. Should the function be non-const? (it modifies state!)
3. Is it a legacy API mismatch? (document and isolate)

---

## Notes

- `std::as_const` is in `<utility>` since C++17.
- Prefer `std::as_const` over `static_cast<const T&>(x)` for readability.
- `const_cast` is the only cast that can remove const/volatile.
- If you need `const_cast` on `this`, use `mutable` members instead.
- `const_cast` to add const is always safe and sometimes useful with templates.
