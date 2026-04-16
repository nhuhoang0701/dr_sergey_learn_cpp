# Use std::hash and define custom hash specializations

**Category:** Standard Library — Utilities  
**Item:** #477  
**Standard:** C++11 / C++17  
**Reference:** <https://en.cppreference.com/w/cpp/utility/hash>  

---

## Topic Overview

`std::hash<T>` is a function object that computes hash values, used by `std::unordered_map`, `std::unordered_set`, etc. The standard provides specializations for built-in types and standard library types. For user-defined types, you must provide your own.

### Standard Specializations

| Type | Provided? |
| --- | --- |
| `int`, `long`, `size_t`, etc. | Yes |
| `float`, `double` | Yes |
| `std::string`, `std::string_view` | Yes |
| `std::shared_ptr<T>`, `std::unique_ptr<T>` | Yes |
| Pointers (`T*`) | Yes |
| User-defined types | **No** — you must specialize |

### Hash Requirements

A good hash function should:

1. Be **deterministic** — same input always gives same output.
2. Have **uniform distribution** — spread values evenly across buckets.
3. Have **low collision rate** — different inputs should rarely hash to the same value.
4. Be **fast** — hashing must be cheaper than the operations it enables.

### Core Syntax

```cpp

#include <functional>
#include <unordered_map>
#include <string>
#include <iostream>

int main() {
    std::hash<int> int_hash;
    std::hash<std::string> str_hash;

    std::cout << "hash(42) = " << int_hash(42) << "\n";
    std::cout << "hash(\"hello\") = " << str_hash("hello") << "\n";

    // Used implicitly by unordered containers
    std::unordered_map<std::string, int> scores;
    scores["Alice"] = 95;  // std::hash<string> called internally
}

```

---

## Self-Assessment

### Q1: Specialize std::hash for a user-defined type to enable use in unordered_map

**Answer:**

```cpp

#include <functional>
#include <unordered_map>
#include <string>
#include <iostream>

struct Employee {
    std::string name;
    int id;

    bool operator==(const Employee& other) const = default;
};

// Method 1: Specialize std::hash in namespace std
template<>
struct std::hash<Employee> {
    std::size_t operator()(const Employee& e) const noexcept {
        std::size_t h1 = std::hash<std::string>{}(e.name);
        std::size_t h2 = std::hash<int>{}(e.id);
        // Combine hashes (see Q2 for proper combining)
        return h1 ^ (h2 << 1);
    }
};

int main() {
    // Now Employee works as an unordered_map key
    std::unordered_map<Employee, double> salaries;
    salaries[{"Alice", 101}] = 75000.0;
    salaries[{"Bob", 102}] = 82000.0;

    Employee query{"Alice", 101};
    if (auto it = salaries.find(query); it != salaries.end()) {
        std::cout << it->first.name << ": $" << it->second << "\n";
    }
    // Output: Alice: $75000

    // Also works with unordered_set
    std::unordered_set<Employee> team;
    team.insert({"Carol", 103});

    // Method 2: pass hash as template parameter (no specialization needed)
    struct EmployeeHash {
        std::size_t operator()(const Employee& e) const noexcept {
            return std::hash<std::string>{}(e.name) ^
                   (std::hash<int>{}(e.id) << 1);
        }
    };
    std::unordered_map<Employee, double, EmployeeHash> alt_map;
}

```

---

### Q2: Use hash combining (e.g., boost::hash_combine pattern) to hash multi-field structs correctly

**Answer:**

```cpp

#include <functional>
#include <unordered_set>
#include <string>
#include <iostream>

// Generic hash_combine: mixes a new hash into an existing seed
// Based on boost::hash_combine — good avalanche properties
inline void hash_combine(std::size_t& seed, std::size_t value) {
    seed ^= value + 0x9e3779b9 + (seed << 6) + (seed >> 2);
}

// Variadic hash helper
template<typename T, typename... Rest>
void hash_all(std::size_t& seed, const T& first, const Rest&... rest) {
    hash_combine(seed, std::hash<T>{}(first));
    (hash_combine(seed, std::hash<Rest>{}(rest)), ...);  // fold expression
}

struct Address {
    std::string street;
    std::string city;
    int zip;

    bool operator==(const Address&) const = default;
};

template<>
struct std::hash<Address> {
    std::size_t operator()(const Address& a) const noexcept {
        std::size_t seed = 0;
        hash_all(seed, a.street, a.city, a.zip);
        return seed;
    }
};

int main() {
    std::unordered_set<Address> addresses;
    addresses.insert({"123 Main St", "Springfield", 62701});
    addresses.insert({"456 Oak Ave", "Shelbyville", 62702});
    addresses.insert({"123 Main St", "Springfield", 62701});  // duplicate

    std::cout << "unique addresses: " << addresses.size() << "\n";
    // Output: unique addresses: 2

    // Why simple XOR is BAD:
    // hash("a") ^ hash("b") == hash("b") ^ hash("a")
    // This means {street="a", city="b"} and {street="b", city="a"}
    // would collide! hash_combine avoids this by being order-dependent.

    // Demonstrate: two different orderings hash differently
    std::size_t s1 = 0, s2 = 0;
    hash_combine(s1, std::hash<int>{}(1));
    hash_combine(s1, std::hash<int>{}(2));

    hash_combine(s2, std::hash<int>{}(2));
    hash_combine(s2, std::hash<int>{}(1));

    std::cout << "hash(1,2) = " << s1 << "\n";
    std::cout << "hash(2,1) = " << s2 << "\n";
    std::cout << "different: " << std::boolalpha << (s1 != s2) << "\n";
    // Output: different: true
}

```

**Why `0x9e3779b9`?** It's the integer part of `φ⁻¹ × 2³²` (golden ratio inverse). This constant provides good bit mixing — ensuring that similar inputs produce very different hash values.

---

### Q3: Explain why a poor hash function causes unordered_map to degrade to O(n) lookup

**Answer:**

`std::unordered_map` uses a **hash table with separate chaining** (linked lists in buckets). Performance depends on how evenly keys distribute across buckets:

| Scenario | Avg lookup | Worst lookup |
| --- | --- | --- |
| Good hash (uniform) | O(1) | O(n/bucket_count) |
| Poor hash (many collisions) | O(n) | O(n) |
| Constant hash (all same) | **O(n)** | **O(n)** |

```cpp

#include <unordered_map>
#include <string>
#include <iostream>
#include <chrono>

struct Point {
    int x, y;
    bool operator==(const Point&) const = default;
};

// TERRIBLE hash: always returns the same value
struct BadHash {
    std::size_t operator()(const Point&) const { return 42; }
};

// GOOD hash: proper combining
struct GoodHash {
    std::size_t operator()(const Point& p) const {
        std::size_t seed = 0;
        seed ^= std::hash<int>{}(p.x) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
        seed ^= std::hash<int>{}(p.y) + 0x9e3779b9 + (seed << 6) + (seed >> 2);
        return seed;
    }
};

int main() {
    constexpr int N = 10000;

    // Bad hash: all elements in one bucket → linked list traversal
    std::unordered_map<Point, int, BadHash> bad_map;
    auto t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        bad_map[{i, i}] = i;
    }
    auto t2 = std::chrono::steady_clock::now();
    auto bad_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    // Good hash: elements spread evenly → near O(1) per operation
    std::unordered_map<Point, int, GoodHash> good_map;
    t1 = std::chrono::steady_clock::now();
    for (int i = 0; i < N; ++i) {
        good_map[{i, i}] = i;
    }
    t2 = std::chrono::steady_clock::now();
    auto good_ms = std::chrono::duration_cast<std::chrono::milliseconds>(t2 - t1).count();

    std::cout << "bad hash:  " << bad_ms  << " ms\n";
    std::cout << "good hash: " << good_ms << " ms\n";
    std::cout << "bad  max_bucket_size: " << bad_map.max_bucket_count() << "\n";

    // Check bucket distribution
    size_t max_bucket = 0;
    for (size_t i = 0; i < bad_map.bucket_count(); ++i) {
        max_bucket = std::max(max_bucket, bad_map.bucket_size(i));
    }
    std::cout << "bad hash largest bucket: " << max_bucket << " elements\n";
    // Output: bad hash largest bucket: 10000 elements (ALL in one bucket!)

    max_bucket = 0;
    for (size_t i = 0; i < good_map.bucket_count(); ++i) {
        max_bucket = std::max(max_bucket, good_map.bucket_size(i));
    }
    std::cout << "good hash largest bucket: " << max_bucket << " elements\n";
    // Output: good hash largest bucket: ~3-5 elements
}

```

**Why O(n):** When all elements hash to the same bucket, every lookup must traverse the entire chain in that bucket — effectively a linear search through an unsorted list of size n.

---

## Notes

- `std::hash` is not required to be deterministic across program runs — the standard allows randomized hashing.
- Always implement `operator==` alongside `std::hash` — unordered containers need both.
- For `std::pair` and `std::tuple`, there is no standard `std::hash` specialization. You must provide your own or use a library.
- In C++17, `std::hash<std::string_view>` was added, compatible with `std::hash<std::string>` for transparent lookup.
- Consider `boost::hash` or `absl::Hash` for production code — they provide better hash combining and type support than manual implementations.

```text
