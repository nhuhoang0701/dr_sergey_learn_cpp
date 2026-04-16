# Implement persistent and immutable data structures in C++

**Category:** Functional Programming Patterns  
**Standard:** C++17  
**Reference:** <https://github.com/arximboldi/immer>  

---

## Topic Overview

Persistent data structures preserve previous versions when modified. Instead of mutating in place, operations return a new version (sharing most of the old structure via structural sharing).

### Why Persistent Data Structures

```cpp

Mutable:     v1 = [1, 2, 3]
             v1.push_back(4)    // v1 is now [1, 2, 3, 4] — old version lost

Persistent:  v1 = [1, 2, 3]
             v2 = v1.push_back(4)  // v1 = [1, 2, 3], v2 = [1, 2, 3, 4]
             // Both versions exist simultaneously, sharing memory

```

### Persistent Vector (using immer library)

```cpp

#include <immer/vector.hpp>
#include <iostream>

int main() {
    auto v1 = immer::vector<int>{1, 2, 3};
    auto v2 = v1.push_back(4);     // v1 unchanged, v2 has 4 elements
    auto v3 = v2.set(0, 99);       // v3 = [99, 2, 3, 4]

    std::cout << "v1: ";
    for (auto x : v1) std::cout << x << " ";  // 1 2 3
    std::cout << "\nv2: ";
    for (auto x : v2) std::cout << x << " ";  // 1 2 3 4
    std::cout << "\nv3: ";
    for (auto x : v3) std::cout << x << " ";  // 99 2 3 4
    std::cout << "\n";

    // Structural sharing: v1, v2, v3 share most of their memory
    // Only the "spine" is copied — O(log N) per operation
}

```

### Simple Persistent List (from scratch)

```cpp

#include <memory>
#include <iostream>

template<typename T>
class PList {
    struct Node {
        T value;
        std::shared_ptr<const Node> next;
    };
    std::shared_ptr<const Node> head_;

public:
    PList() = default;
    PList(T value, PList tail)
        : head_(std::make_shared<Node>(Node{std::move(value), tail.head_})) {}

    PList prepend(T value) const {
        return PList(std::move(value), *this);
    }

    bool empty() const { return !head_; }
    const T& front() const { return head_->value; }
    PList tail() const { return PList{head_->next}; }

private:
    PList(std::shared_ptr<const Node> h) : head_(std::move(h)) {}
};

int main() {
    auto l1 = PList<int>{}.prepend(3).prepend(2).prepend(1);
    auto l2 = l1.prepend(0);  // l1 is [1,2,3], l2 is [0,1,2,3]
    // l2 shares l1's nodes — zero copying

    for (auto l = l2; !l.empty(); l = l.tail())
        std::cout << l.front() << " ";  // 0 1 2 3
}

```

---

## Self-Assessment

### Q1: What is structural sharing and why is it efficient

When creating a new version of a data structure, only the changed nodes are copied; unchanged subtrees are shared between old and new versions via shared pointers. A balanced tree of N elements shares all but O(log N) nodes, making updates O(log N) in time and space.

### Q2: When are persistent data structures useful in C++

- Undo/redo systems (each state is a version)
- Concurrent programming (immutable data needs no locks)
- Functional reactive programming
- Transactional memory (commit/rollback)
- Time-travel debugging (replay from any state)

### Q3: What's the performance trade-off vs mutable structures

Persistent structures add O(log32 N) overhead per access (radix-balanced trees). For N < 1 million, this is nearly constant. The trade-off: slightly slower element access, but free snapshotting, thread safety without locks, and no iterator invalidation.

---

## Notes

- The **immer** library provides production-quality persistent vectors, maps, and sets for C++.
- Persistent structures use reference counting (`shared_ptr`) for memory management.
- Hash-Array Mapped Tries (HAMT) are the most common implementation (used by Clojure, Scala, immer).
- Copy-on-write (CoW) is a simpler but less efficient form of persistence.
