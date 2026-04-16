# Use priority_queue, stack, and queue as container adaptors

**Category:** Standard Library — Containers  
**Item:** #69  
**Reference:** <https://en.cppreference.com/w/cpp/container/priority_queue>  

---

## Topic Overview

Container adaptors **wrap** an underlying container to provide a restricted interface. They are not containers themselves — they have no iterators, no `begin()`/`end()`, and no direct element access. They enforce a specific access pattern.

### The Three Adaptors

| Adaptor | Access pattern | Default underlying | Operations |
| --- | --- | --- | --- |
| `std::stack<T>` | LIFO (Last In, First Out) | `std::deque<T>` | `push`, `pop`, `top` |
| `std::queue<T>` | FIFO (First In, First Out) | `std::deque<T>` | `push`, `pop`, `front`, `back` |
| `std::priority_queue<T>` | Largest first (max-heap) | `std::vector<T>` | `push`, `pop`, `top` |

### Underlying Container Requirements

```cpp

stack needs:     back(), push_back(), pop_back()
                 → works with: vector, deque, list

queue needs:     front(), back(), push_back(), pop_front()
                 → works with: deque, list
                 → NOT vector (no pop_front)

priority_queue:  front(), push_back(), pop_back() + random access iterators
                 → works with: vector, deque
                 → NOT list (no random access)

```

### Core Usage

```cpp

#include <iostream>
#include <stack>
#include <queue>
#include <vector>
#include <deque>

int main() {
    // === Stack (LIFO) ===
    std::stack<int> stk;
    stk.push(1); stk.push(2); stk.push(3);
    // Internal: [1, 2, 3]  ← top is 3
    std::cout << "Stack top: " << stk.top() << "\n";  // 3
    stk.pop();
    std::cout << "After pop: " << stk.top() << "\n";  // 2

    // === Queue (FIFO) ===
    std::queue<int> q;
    q.push(1); q.push(2); q.push(3);
    // Internal: [1, 2, 3]  front=1, back=3
    std::cout << "Queue front: " << q.front() << "\n";  // 1
    q.pop();
    std::cout << "After pop: " << q.front() << "\n";    // 2

    // === Priority Queue (max-heap) ===
    std::priority_queue<int> pq;
    pq.push(30); pq.push(10); pq.push(50); pq.push(20);
    // Heap: 50 at top
    std::cout << "PQ top: " << pq.top() << "\n";  // 50
    pq.pop();
    std::cout << "After pop: " << pq.top() << "\n";  // 30

    // Min-heap with std::greater:
    std::priority_queue<int, std::vector<int>, std::greater<int>> min_pq;
    min_pq.push(30); min_pq.push(10); min_pq.push(50);
    std::cout << "Min-PQ top: " << min_pq.top() << "\n";  // 10

    return 0;
}

```

### Important Notes

- All three have `push()`, `pop()`, `top()` (or `front()`). `pop()` returns void (not the element!).
- `top()` and `front()` on an empty adaptor is **undefined behavior**.
- Priority queue internally uses `std::make_heap`, `std::push_heap`, `std::pop_heap`.
- C++23 adds range constructors for all three adaptors.

---

## Self-Assessment

### Q1: Implement Dijkstra's shortest path using std::priority_queue with a custom comparator

```cpp

#include <iostream>
#include <vector>
#include <queue>
#include <limits>
#include <functional>

int main() {
    // === Graph as adjacency list ===
    // Node → vector of (neighbor, weight)
    constexpr int N = 6;  // Nodes: 0..5
    using Edge = std::pair<int, int>;  // (neighbor, weight)
    std::vector<std::vector<Edge>> graph(N);

    // Build graph (undirected)
    auto add_edge = [&](int u, int v, int w) {
        graph[u].push_back({v, w});
        graph[v].push_back({u, w});
    };

    //     0 ---4--- 1 ---2--- 2
    //     |         |         |
    //     2         1         3
    //     |         |         |
    //     3 ---3--- 4 ---5--- 5
    add_edge(0, 1, 4);
    add_edge(0, 3, 2);
    add_edge(1, 2, 2);
    add_edge(1, 4, 1);
    add_edge(2, 5, 3);
    add_edge(3, 4, 3);
    add_edge(4, 5, 5);

    // === Dijkstra with priority_queue ===
    int source = 0;
    std::vector<int> dist(N, std::numeric_limits<int>::max());
    std::vector<int> parent(N, -1);

    // Min-heap: (distance, node)
    // std::greater makes it a min-priority_queue (smallest distance first)
    using PQEntry = std::pair<int, int>;  // (dist, node)
    std::priority_queue<PQEntry, std::vector<PQEntry>, std::greater<PQEntry>> pq;

    dist[source] = 0;
    pq.push({0, source});

    while (!pq.empty()) {
        auto [d, u] = pq.top();
        pq.pop();

        // Skip stale entries (already found shorter path)
        if (d > dist[u]) continue;

        for (auto [v, w] : graph[u]) {
            int new_dist = dist[u] + w;
            if (new_dist < dist[v]) {
                dist[v] = new_dist;
                parent[v] = u;
                pq.push({new_dist, v});  // Lazy insertion (may add duplicates)
            }
        }
    }

    // === Print results ===
    std::cout << "Shortest distances from node " << source << ":\n";
    for (int i = 0; i < N; ++i) {
        std::cout << "  Node " << i << ": dist=" << dist[i] << "  path: ";

        // Reconstruct path
        std::vector<int> path;
        for (int v = i; v != -1; v = parent[v])
            path.push_back(v);
        for (int j = static_cast<int>(path.size()) - 1; j >= 0; --j) {
            std::cout << path[j];
            if (j > 0) std::cout << " → ";
        }
        std::cout << "\n";
    }

    // Expected output:
    // Node 0: dist=0  path: 0
    // Node 1: dist=4  path: 0 → 1
    // Node 2: dist=6  path: 0 → 1 → 2
    // Node 3: dist=2  path: 0 → 3
    // Node 4: dist=5  path: 0 → 3 → 4  (or 0→1→4)
    // Node 5: dist=9  path: 0 → 1 → 2 → 5

    return 0;
}

```

**How it works:**

- Dijkstra's algorithm greedily processes the nearest unvisited node. A **min-priority_queue** efficiently retrieves the node with the smallest tentative distance.
- `std::priority_queue<pair<int,int>, vector<...>, std::greater<...>>` creates a min-heap where the smallest `(distance, node)` pair is at the top.
- **Lazy deletion:** Instead of decreasing keys (which `std::priority_queue` doesn't support), we push new entries and skip stale ones with `if (d > dist[u]) continue`. This is clean and efficient.
- Time complexity: O((V + E) log V) with the priority queue.
- The `std::greater<>` comparator compares `pair` lexicographically: first by distance, then by node number — exactly what we need.

### Q2: Show how to change the underlying container of a priority_queue from deque to vector

```cpp

#include <iostream>
#include <queue>
#include <vector>
#include <deque>
#include <list>
#include <functional>
#include <string>

int main() {
    // === priority_queue template: ===
    // template<class T, class Container = vector<T>,
    //          class Compare = less<typename Container::value_type>>

    // Default: uses vector<int>, max-heap
    std::priority_queue<int> default_pq;  // vector<int>, less<int>

    // === Explicit vector (same as default) ===
    std::priority_queue<int, std::vector<int>, std::less<int>> vec_pq;

    // === Use deque as underlying container ===
    std::priority_queue<int, std::deque<int>> deque_pq;

    // === Use deque + custom comparator (min-heap) ===
    std::priority_queue<int, std::deque<int>, std::greater<int>> deque_min_pq;

    // === Populate all and compare behavior ===
    int data[] = {30, 10, 50, 20, 40};
    for (int x : data) {
        vec_pq.push(x);
        deque_pq.push(x);
        deque_min_pq.push(x);
    }

    std::cout << "vector<int> max-heap top:  " << vec_pq.top() << "\n";    // 50
    std::cout << "deque<int> max-heap top:   " << deque_pq.top() << "\n";   // 50
    std::cout << "deque<int> min-heap top:   " << deque_min_pq.top() << "\n"; // 10

    // === Stack with different containers ===
    std::stack<int, std::vector<int>> vec_stack;
    std::stack<int, std::deque<int>>  deq_stack;  // default
    std::stack<int, std::list<int>>   list_stack;

    vec_stack.push(1); vec_stack.push(2);
    deq_stack.push(1); deq_stack.push(2);
    list_stack.push(1); list_stack.push(2);

    std::cout << "\nvector stack top:  " << vec_stack.top() << "\n";  // 2
    std::cout << "deque stack top:   " << deq_stack.top() << "\n";    // 2
    std::cout << "list stack top:    " << list_stack.top() << "\n";   // 2

    // === Queue with different containers ===
    std::queue<int, std::deque<int>> deq_queue;   // default
    std::queue<int, std::list<int>>  list_queue;
    // std::queue<int, std::vector<int>> vec_queue; // ERROR: vector has no pop_front!

    std::cout << "\nContainer requirements summary:\n";
    std::cout << "  priority_queue: random access + push/pop_back\n";
    std::cout << "    → vector (default, best cache locality)\n";
    std::cout << "    → deque (ok, but worse cache behavior)\n";
    std::cout << "    → list: NO (no random access)\n";
    std::cout << "  stack: push/pop/back\n";
    std::cout << "    → deque (default), vector, list all work\n";
    std::cout << "  queue: push_back + pop_front + front\n";
    std::cout << "    → deque (default), list work\n";
    std::cout << "    → vector: NO (no efficient pop_front)\n";

    return 0;
}

```

**How it works:**

- The second template parameter of each adaptor specifies the underlying container. You can swap it to any container that satisfies the required operations.
- `priority_queue` defaults to `vector` because heap operations (`push_heap`, `pop_heap`) require random access iterators and `vector` has the best cache locality for heap traversal.
- `deque` works for `priority_queue` but is slightly slower because deque's non-contiguous memory hurts cache performance during heap sift operations.
- `stack` and `queue` default to `deque` because deque efficiently supports both front and back operations. For stack-only usage, `vector` may be slightly faster.

### Q3: Explain why these adaptors do not expose iterators and what that implies for algorithm use

```cpp

#include <iostream>
#include <stack>
#include <queue>
#include <vector>
#include <algorithm>

int main() {
    // === Container adaptors have NO iterators ===
    std::stack<int> stk;
    stk.push(1); stk.push(2); stk.push(3);

    // These do NOT compile:
    // for (auto it = stk.begin(); it != stk.end(); ++it) ...  // ERROR
    // std::sort(stk.begin(), stk.end());                       // ERROR
    // std::find(stk.begin(), stk.end(), 2);                    // ERROR
    // for (int x : stk) ...                                     // ERROR

    // === Why no iterators? ===
    // Adaptors enforce an ACCESS PATTERN:
    //   stack:          only access top
    //   queue:          only access front/back
    //   priority_queue: only access top (max/min element)
    //
    // Exposing iterators would let you:
    //   - Read elements in wrong order (violating LIFO/FIFO/heap)
    //   - Modify elements (breaking the heap invariant)
    //   - Sort or shuffle (meaningless for a queue)
    // This would defeat the purpose of the adaptor.

    // === Implications for algorithms ===
    std::cout << "What you CANNOT do:\n";
    std::cout << "  - std::find, std::count on a stack/queue\n";
    std::cout << "  - std::sort, std::reverse on a priority_queue\n";
    std::cout << "  - Range-based for loop\n";
    std::cout << "  - std::copy to an output iterator\n\n";

    // === Workaround 1: Drain the adaptor ===
    std::cout << "Drain stack: ";
    while (!stk.empty()) {
        std::cout << stk.top() << " ";  // 3 2 1 (LIFO order)
        stk.pop();
    }
    std::cout << "\n";

    // === Workaround 2: Access underlying container (non-portable) ===
    // The underlying container is a protected member 'c'
    // You can access it via inheritance trick:
    struct HackStack : std::stack<int> {
        using std::stack<int>::c;  // Expose protected member
    };

    HackStack& hack = static_cast<HackStack&>(stk);
    stk.push(10); stk.push(20); stk.push(30);
    std::cout << "Underlying container: ";
    for (int x : hack.c) std::cout << x << " ";  // 10 20 30
    std::cout << "\n";
    // WARNING: This is technically valid C++ but poor practice.

    // === Workaround 3: Use the underlying container directly ===
    // If you need iteration, just use vector/deque directly
    std::vector<int> heap_data = {30, 10, 50, 20, 40};
    std::make_heap(heap_data.begin(), heap_data.end());
    // Now heap_data IS a heap AND you can iterate it!

    std::cout << "\nHeap (via vector + make_heap): ";
    for (int x : heap_data) std::cout << x << " ";
    std::cout << "\n";  // 50 40 30 20 10 (heap order, not sorted!)

    std::cout << "Top: " << heap_data.front() << "\n";  // 50

    // Pop from heap:
    std::pop_heap(heap_data.begin(), heap_data.end());
    heap_data.pop_back();
    std::cout << "After pop, top: " << heap_data.front() << "\n";  // 40

    // === When to use adaptors vs raw containers ===
    std::cout << "\nUse adaptors when:\n";
    std::cout << "  - You want to enforce LIFO/FIFO/priority access\n";
    std::cout << "  - You want to prevent accidental misuse\n";
    std::cout << "  - Interface clarity > flexibility\n";
    std::cout << "\nUse raw containers when:\n";
    std::cout << "  - You need to iterate or search elements\n";
    std::cout << "  - You need algorithms (find, sort, copy)\n";
    std::cout << "  - You need both ends AND middle access\n";

    return 0;
}

```

**How it works:**

- Container adaptors deliberately omit `begin()`, `end()`, and iterators to **enforce their access discipline**. A stack should only be accessed from the top; a priority queue should only expose the highest-priority element.
- This means **no standard algorithm** (`std::find`, `std::sort`, `std::count`, etc.) works with adaptors, because algorithms require iterator ranges.
- **If you need iteration + priority access:** Use a `std::vector` with `std::make_heap`/`push_heap`/`pop_heap` — you get both heap operations AND the ability to iterate/search the underlying data.
- The protected member `c` can be accessed via inheritance, but this breaks the abstraction the adaptor provides and should be avoided in production code.
- The no-iterator design is intentional: it prevents bugs like modifying a priority queue element in-place (which would break the heap invariant).

---

## Notes

- **`pop()` returns void:** This is a deliberate design choice. Returning the value would require an extra copy/move (exception-unsafe). Always call `top()` before `pop()`.
- **Custom comparators:** Use a lambda or functor as the third template parameter of `priority_queue`. With C++20, you can use a lambda directly: `priority_queue<int, vector<int>, decltype([](int a, int b){ return a > b; })>`.
- **`std::priority_queue` is NOT a sorted container.** It's a heap — only the top element is guaranteed to be the max (or min). Internal order is unspecified.
- **Thread safety:** Adaptors have the same thread-safety guarantees as their underlying container — none for concurrent modification. Wrap with a mutex if sharing across threads.
- **C++23:** Adds `std::flat_set` and `std::flat_map` — sorted adaptors over `std::vector` that DO provide iterators. These are the "fixed" version of what container adaptors could have been.
