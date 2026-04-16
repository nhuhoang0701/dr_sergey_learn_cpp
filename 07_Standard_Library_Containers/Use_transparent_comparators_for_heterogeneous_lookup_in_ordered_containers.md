# Use transparent comparators for heterogeneous lookup in ordered containers

**Category:** Standard Library — Containers  
**Item:** #349  
**Standard:** C++14 (introduced), C++17 (refined)  
**Reference:** <https://en.cppreference.com/w/cpp/utility/functional/less_void>  

---

## Topic Overview

**Heterogeneous lookup** allows you to search an ordered container (like `set` or `map`) using a key type **different** from the container's key type — without constructing a temporary key object. This is enabled by **transparent comparators** like `std::less<>`.

### The Problem

```cpp

std::set<std::string> names = {"Alice", "Bob", "Charlie"};

// Without transparent comparator:
names.find("Bob");  // "Bob" is const char* → constructs temporary std::string → heap allocation!
// Equivalent to: names.find(std::string("Bob"));

```

### The Solution

```cpp

std::set<std::string, std::less<>> names = {"Alice", "Bob", "Charlie"};
//                     ^^^^^^^^^^^ transparent comparator

names.find("Bob");           // finds directly with const char* — NO temporary string!
names.find(std::string_view("Bob"));  // works too — NO allocation!

// How? std::less<> defines operator()(const A&, const B&) for ALL types
// It just calls a < b, which works for string vs string_view, string vs const char*, etc.

```

### How It Works

```cpp

// std::less<std::string> (default, non-transparent):
struct less_string {
    bool operator()(const std::string& a, const std::string& b) const;
    // ONLY accepts std::string — forces conversion of other types
};

// std::less<> (transparent):
struct less_void {
    using is_transparent = void;  // ← this tag type enables heterogeneous lookup
    
    template <typename T, typename U>
    bool operator()(const T& a, const U& b) const {
        return a < b;  // works for ANY comparable types
    }
};

```

The container checks for `is_transparent` tag in the comparator. If present, `find()`, `count()`, `lower_bound()`, `upper_bound()`, `equal_range()`, and `contains()` get **template overloads** that accept any comparable type.

---

## Self-Assessment

### Q1: Use std::less<> in a std::set<std::string> to enable lookup by string_view

```cpp

#include <iostream>
#include <set>
#include <string>
#include <string_view>

// === Custom allocator to detect string constructions ===
static int string_constructions = 0;

struct CountingString : std::string {
    using std::string::string;
    CountingString(const char* s) : std::string(s) { ++string_constructions; }
    CountingString(std::string_view sv) : std::string(sv) { ++string_constructions; }
};

int main() {
    // === Without transparent comparator ===
    {
        std::set<std::string> names = {"Alice", "Bob", "Charlie", "Diana"};

        // Every find() with a non-string creates a temporary std::string
        auto it1 = names.find("Bob");          // constructs temporary string "Bob"
        auto it2 = names.find("Charlie");      // constructs temporary string "Charlie"
        // 2 unnecessary heap allocations (assuming no SSO)
        std::cout << "Found: " << (it1 != names.end()) << ", " << (it2 != names.end()) << "\n";
    }

    // === With transparent comparator ===
    {
        std::set<std::string, std::less<>> names = {"Alice", "Bob", "Charlie", "Diana"};
        //                    ^^^^^^^^^^^ transparent

        // find() now accepts ANY type that is comparable with std::string
        auto it1 = names.find("Bob");                              // const char* — no temp
        auto it2 = names.find(std::string_view("Charlie"));        // string_view — no temp
        std::cout << "Found: " << (it1 != names.end()) << ", " << (it2 != names.end()) << "\n";

        // count() and contains() also work heterogeneously:
        std::cout << "Has Diana: " << names.contains("Diana") << "\n";     // C++20
        std::cout << "Count Bob: " << names.count(std::string_view("Bob")) << "\n";

        // lower_bound / upper_bound for range queries:
        auto lb = names.lower_bound("B");
        auto ub = names.lower_bound("D");
        std::cout << "Names starting with B or C: ";
        for (auto it = lb; it != ub; ++it)
            std::cout << *it << " ";
        std::cout << "\n";
        // Bob Charlie
    }

    return 0;
}

```

**How it works:**

- `std::less<>` is a transparent comparator: its `operator()` is a template that accepts any comparable types.
- When the container's comparator has an `is_transparent` tag, `find()` and related functions get template overloads that accept non-key types.
- `const char*` and `std::string_view` are directly comparable with `std::string` via `operator<`, so no temporary `std::string` is constructed.
- This avoids heap allocations and is measurably faster for string-keyed containers in hot paths.

### Q2: Show that lookup with a string literal on a set<std::string, std::less<>> avoids constructing a string

```cpp

#include <iostream>
#include <set>
#include <string>
#include <cstdlib>

// === Track heap allocations ===
static int alloc_count = 0;
void* operator new(std::size_t sz) {
    ++alloc_count;
    return std::malloc(sz);
}
void operator delete(void* p) noexcept { std::free(p); }
void operator delete(void* p, std::size_t) noexcept { std::free(p); }

int main() {
    // === Setup: strings long enough to avoid SSO ===
    // (SSO = Small String Optimization: short strings don't allocate)
    std::string long1 = "this_is_a_very_long_string_alice_12345678";
    std::string long2 = "this_is_a_very_long_string_bob_1234567890";
    std::string long3 = "this_is_a_very_long_string_charlie_123456";

    // === Non-transparent set ===
    {
        std::set<std::string> s;
        s.insert(long1);
        s.insert(long2);
        s.insert(long3);

        alloc_count = 0;
        auto it = s.find("this_is_a_very_long_string_bob_1234567890");
        std::cout << "Non-transparent find: " << alloc_count << " allocations";
        std::cout << " (found=" << (it != s.end()) << ")\n";
        // Output: 1 allocation — the const char* was converted to std::string
    }

    // === Transparent set ===
    {
        std::set<std::string, std::less<>> s;
        s.insert(long1);
        s.insert(long2);
        s.insert(long3);

        alloc_count = 0;
        auto it = s.find("this_is_a_very_long_string_bob_1234567890");
        std::cout << "Transparent find:     " << alloc_count << " allocations";
        std::cout << " (found=" << (it != s.end()) << ")\n";
        // Output: 0 allocations — const char* compared directly via operator<
    }

    // === Also works with map ===
    {
        std::map<std::string, int, std::less<>> m;
        m["alpha"] = 1;
        m["beta"] = 2;
        m["gamma"] = 3;

        alloc_count = 0;
        auto it = m.find("beta");
        std::cout << "Transparent map find: " << alloc_count << " allocations";
        std::cout << " (value=" << (it != m.end() ? it->second : -1) << ")\n";
        // Output: 0 allocations
    }

    return 0;
}

```

**How it works:**

- Without `std::less<>`, `find("literal")` must convert `const char*` → `std::string` (triggering heap allocation for strings longer than SSO threshold, typically 15-22 chars).
- With `std::less<>`, the comparator directly compares `const char*` against `std::string` using `operator<`, which does character-by-character comparison without allocation.
- This is especially impactful in hot loops doing many lookups with string literals or `string_view`.

### Q3: Implement a custom transparent comparator for a set of user-defined objects

```cpp

#include <iostream>
#include <set>
#include <string>
#include <string_view>

struct Employee {
    int id;
    std::string name;
    std::string department;
};

// === Custom transparent comparator ===
// Allows lookup by int (id), string_view (name), or Employee
struct EmployeeCompare {
    using is_transparent = void;  // ← REQUIRED: enables heterogeneous lookup

    // Employee vs Employee: compare by id
    bool operator()(const Employee& a, const Employee& b) const {
        return a.id < b.id;
    }

    // Employee vs int: lookup by id
    bool operator()(const Employee& e, int id) const {
        return e.id < id;
    }
    bool operator()(int id, const Employee& e) const {
        return id < e.id;
    }

    // Employee vs string_view: lookup by name
    // (Note: this only works correctly if the set is sorted by name!)
    // For this example, we sort by id, so name lookup would require a different approach.
};

// === Alternative: name-based comparator ===
struct ByName {
    using is_transparent = void;

    bool operator()(const Employee& a, const Employee& b) const {
        return a.name < b.name;
    }
    bool operator()(const Employee& e, std::string_view name) const {
        return e.name < name;
    }
    bool operator()(std::string_view name, const Employee& e) const {
        return name < e.name;
    }
};

int main() {
    // === Set sorted by ID, lookup by int ===
    std::set<Employee, EmployeeCompare> by_id;
    by_id.insert({101, "Alice", "Engineering"});
    by_id.insert({205, "Bob", "Marketing"});
    by_id.insert({150, "Charlie", "Engineering"});

    // Heterogeneous lookup by int — no Employee constructed!
    auto it = by_id.find(205);
    if (it != by_id.end())
        std::cout << "Found by id 205: " << it->name << " (" << it->department << ")\n";
    // Output: Found by id 205: Bob (Marketing)

    std::cout << "Contains id 999: " << by_id.contains(999) << "\n";  // 0

    // === Set sorted by name, lookup by string_view ===
    std::set<Employee, ByName> by_name;
    by_name.insert({101, "Alice", "Engineering"});
    by_name.insert({205, "Bob", "Marketing"});
    by_name.insert({150, "Charlie", "Engineering"});

    // Heterogeneous lookup by string_view — no Employee constructed!
    auto it2 = by_name.find(std::string_view("Charlie"));
    if (it2 != by_name.end())
        std::cout << "Found by name: " << it2->name << " (id=" << it2->id << ")\n";
    // Output: Found by name: Charlie (id=150)

    // Range query: all names starting with "A" or "B"
    auto lb = by_name.lower_bound(std::string_view("A"));
    auto ub = by_name.lower_bound(std::string_view("C"));
    std::cout << "Names A-B: ";
    for (auto it = lb; it != ub; ++it)
        std::cout << it->name << " ";
    std::cout << "\n";  // Alice Bob

    return 0;
}

```

**How it works:**

- A custom transparent comparator must:
  1. Define `using is_transparent = void;` (the tag type)
  2. Provide `operator()` overloads for the container's key type AND the lookup type, ensuring consistent ordering
- Both `(Key, LookupType)` and `(LookupType, Key)` overloads are needed
- The ordering must be consistent: if `comp(a, b) == true`, then `comp(b, a) == false`
- This pattern is useful for looking up objects by a single member (id, name) without constructing the full object

---

## Notes

- **`std::less<>`** is the standard transparent comparator. Also available: `std::greater<>`, `std::less_equal<>`, etc.
- **Why not default?** Transparent lookup was not made the default to avoid breaking existing code where types have surprising comparison operators.
- **C++20 `contains()`** also benefits from transparent comparators — `s.contains("key")` avoids allocation.
- **`std::map` too:** `std::map<std::string, int, std::less<>>` enables heterogeneous `find`, `count`, `contains`, etc.
- **Unordered containers (C++20):** Require `transparent_key_equal` with both `std::hash` specializations and `is_transparent` on the equal functor. More complex than ordered containers.
- **Perf impact:** In benchmarks, heterogeneous lookup on `set<string>` can be 2-5× faster for `find()` heavy workloads due to avoided allocations.
