# Use std::list::splice for O(1) node transfers between lists

**Category:** Standard Library — Containers  
**Item:** #347  
**Reference:** <https://en.cppreference.com/w/cpp/container/list/splice>  

---

## Topic Overview

`std::list::splice` transfers elements between lists by **relinking internal node pointers** — no copying, no moving, no allocation. It's O(1) for transferring a single element or an entire list (O(n) only for a subrange if the implementation needs to update `size()`).

### splice() Overloads

```cpp

// 1. Move entire list
void splice(const_iterator pos, list& other);
// Moves ALL elements from 'other' into *this before 'pos'.
// 'other' becomes empty. O(1).

// 2. Move single element
void splice(const_iterator pos, list& other, const_iterator it);
// Moves element at 'it' from 'other' into *this before 'pos'. O(1).

// 3. Move range
void splice(const_iterator pos, list& other,
            const_iterator first, const_iterator last);
// Moves [first, last) from 'other' into *this before 'pos'.
// O(1) if &other == this, otherwise O(distance(first,last))
// because size() of both lists must be updated.

```

### How It Works Internally

```cpp

Before splice(pos, other, it):

this:  [A] ←→ [B] ←→ [C]     pos = iterator to C
other: [X] ←→ [Y] ←→ [Z]     it  = iterator to Y

Step 1: Unlink Y from other:  [X] ←→ [Z]
Step 2: Link Y before C:      [A] ←→ [B] ←→ [Y] ←→ [C]

Result:
this:  [A] ←→ [B] ←→ [Y] ←→ [C]
other: [X] ←→ [Z]

Zero allocation. Zero copy. Just 4 pointer reassignments.

```

### Core Example

```cpp

#include <iostream>
#include <list>
#include <string>

void print(const std::string& label, const std::list<int>& lst) {
    std::cout << label << ": ";
    for (int x : lst) std::cout << x << " ";
    std::cout << "(size=" << lst.size() << ")\n";
}

int main() {
    std::list<int> a = {1, 2, 3, 4, 5};
    std::list<int> b = {10, 20, 30};

    print("a", a);  // a: 1 2 3 4 5 (size=5)
    print("b", b);  // b: 10 20 30 (size=3)

    // Splice entire list b before position 3 in a
    auto it3 = std::next(a.begin(), 2);  // points to 3
    a.splice(it3, b);
    print("\nAfter a.splice(it3, b):", a);  // 1 2 10 20 30 3 4 5
    print("b after splice:", b);             // (empty)

    // Splice single element: move 30 to front
    auto it30 = std::find(a.begin(), a.end(), 30);
    a.splice(a.begin(), a, it30);  // splice within same list
    print("\nAfter moving 30 to front:", a);  // 30 1 2 10 20 3 4 5

    return 0;
}

```

### Important Notes

- **Iterator validity:** Iterators to spliced elements remain valid — they just point to elements now in a different container (or different position).
- **Self-splice:** You can splice within the same list (rearrange elements).
- **size() consideration:** Pre-C++11, `list::size()` could be O(n). Since C++11, `size()` must be O(1), so range-splice must count elements to update the sizes of both lists.

---

## Self-Assessment

### Q1: Splice a subrange from one list into another at an iterator position in O(1)

```cpp

#include <iostream>
#include <list>
#include <string>

void print(const std::string& label, const std::list<int>& lst) {
    std::cout << label << ": ";
    for (int x : lst) std::cout << x << " ";
    std::cout << "(size=" << lst.size() << ")\n";
}

int main() {
    std::list<int> source = {10, 20, 30, 40, 50, 60};
    std::list<int> dest   = {1, 2, 3};

    print("source", source);  // 10 20 30 40 50 60
    print("dest  ", dest);    // 1 2 3

    // Define the subrange to splice: [30, 40, 50)
    auto first = std::next(source.begin(), 2);  // points to 30
    auto last  = std::next(source.begin(), 5);  // points to 60 (past 50)

    // Splice into dest before element 3
    auto pos = std::next(dest.begin(), 2);  // points to 3
    dest.splice(pos, source, first, last);

    print("\ndest after splice ", dest);     // 1 2 30 40 50 3
    print("source after splice", source);   // 10 20 60

    // === Single element splice ===
    // Move element 60 from source to front of dest
    auto it60 = std::find(source.begin(), source.end(), 60);
    dest.splice(dest.begin(), source, it60);

    print("\ndest after single splice ", dest);   // 60 1 2 30 40 50 3
    print("source after single splice", source);  // 10 20

    // === Entire list splice ===
    dest.splice(dest.end(), source);  // Append all remaining source

    print("\ndest after full splice ", dest);   // 60 1 2 30 40 50 3 10 20
    print("source after full splice", source);  // (empty)

    // === Self-splice: rearrange within same list ===
    // Move element '1' to the end
    auto it1 = std::find(dest.begin(), dest.end(), 1);
    dest.splice(dest.end(), dest, it1);
    print("\nAfter moving 1 to end", dest);  // 60 2 30 40 50 3 10 20 1

    return 0;
}

```

**How it works:**

- `splice(pos, other, first, last)` unlinks nodes `[first, last)` from `other` and inserts them before `pos` in `*this`. No elements are copied, moved, or destroyed — only internal pointers are reassigned.
- For **single element** splice: O(1) always.
- For **entire list** splice: O(1) — just reassign head/tail pointers and add sizes.
- For **subrange** splice: O(distance(first, last)) in C++11+ because both lists' `size()` must be updated (the implementation counts the transferred elements). The actual node relinking is still O(1).
- Iterators to the spliced elements remain valid and now refer to positions in the destination list.

### Q2: Explain why splice invalidates the spliced iterators in the source but not the destination

```cpp

#include <iostream>
#include <list>

int main() {
    std::list<int> a = {1, 2, 3, 4, 5};
    std::list<int> b = {10, 20, 30};

    // Save iterators
    auto it_2 = std::next(a.begin(), 1);  // points to 2 in list a
    auto it_4 = std::next(a.begin(), 3);  // points to 4 in list a
    auto it_20 = std::next(b.begin(), 1); // points to 20 in list b

    std::cout << "Before splice:\n";
    std::cout << "  *it_2  = " << *it_2  << " (in list a)\n";  // 2
    std::cout << "  *it_4  = " << *it_4  << " (in list a)\n";  // 4
    std::cout << "  *it_20 = " << *it_20 << " (in list b)\n";  // 20

    // Splice element 20 from b into a (before position it_4)
    a.splice(it_4, b, it_20);
    // a is now: 1 2 3 20 4 5
    // b is now: 10 30

    std::cout << "\nAfter splicing 20 from b to a:\n";

    // it_20 is STILL VALID — it now refers to 20 in list a
    std::cout << "  *it_20 = " << *it_20 << " (now in list a!)\n";  // 20

    // it_2 and it_4 are STILL VALID — they weren't moved
    std::cout << "  *it_2  = " << *it_2 << "\n";   // 2
    std::cout << "  *it_4  = " << *it_4 << "\n";   // 4

    // Verify: traversal proves the new order
    std::cout << "\nList a: ";
    for (int x : a) std::cout << x << " ";
    std::cout << "\n";
    // Output: 1 2 3 20 4 5

    // Verify: iterators reflect new neighbors
    std::cout << "Next after 20: " << *std::next(it_20) << "\n";  // 4
    std::cout << "Prev of 20:    " << *std::prev(it_20) << "\n";  // 3

    // === Key insight ===
    // Splice does NOT "invalidate" iterators in the traditional sense.
    // ALL iterators to spliced elements REMAIN VALID.
    // They just belong to a different container now.
    //
    // The "invalidation" in the source means:
    //   - The element is no longer IN the source container
    //   - Using source.end() on the iterator would be wrong
    //   - But dereferencing the iterator still works — it points to the same node

    // === Practical consequence: iterator-based tracking ===
    // You can store iterators as "handles" to elements:
    std::list<int> tasks = {100, 200, 300};
    auto handle = std::next(tasks.begin());  // handle to element 200

    std::list<int> done;
    done.splice(done.end(), tasks, handle);
    // 'handle' still valid, now points to 200 in 'done'
    std::cout << "\nHandle still works: " << *handle << "\n";  // 200

    return 0;
}

```

**How it works:**

- **Splice does NOT invalidate any iterators.** All iterators to spliced elements remain valid. The C++ standard says: "No iterators or references are invalidated."
- What changes is which container the element belongs to. An iterator that used to refer to an element in `other` now refers to the same element in `*this` — the node is the same heap object, just linked into a different list.
- This is because `list` nodes are independently allocated. Splicing just relinks prev/next pointers — the node's memory address doesn't change, so iterators (which are wrappers around node pointers) remain valid.
- This property makes `std::list` ideal for data structures where you need to move elements between collections while maintaining handles/references to them.

### Q3: Show a task scheduler using list::splice for efficient priority queue promotion

```cpp

#include <iostream>
#include <list>
#include <string>
#include <unordered_map>

// === Task Scheduler with Priority Promotion ===
// Tasks live in priority lanes (lists). Promoting a task = splice to higher lane.
// No allocation, no copy — just O(1) pointer relinking.

struct Task {
    int id;
    std::string name;
    int priority;  // 0 = low, 1 = medium, 2 = high
};

class Scheduler {
    static constexpr int NUM_PRIORITIES = 3;
    std::list<Task> lanes[NUM_PRIORITIES];  // [0]=low, [1]=medium, [2]=high

    // Map task_id → iterator (for O(1) lookup)
    using TaskIter = std::list<Task>::iterator;
    std::unordered_map<int, TaskIter> task_map;
    // Also track which lane each task is in:
    std::unordered_map<int, int> task_lane;

public:
    void add_task(int id, const std::string& name, int priority) {
        lanes[priority].push_back({id, name, priority});
        auto it = std::prev(lanes[priority].end());
        task_map[id] = it;
        task_lane[id] = priority;
        std::cout << "Added task " << id << " '" << name
                  << "' at priority " << priority << "\n";
    }

    void promote(int id) {
        auto map_it = task_map.find(id);
        if (map_it == task_map.end()) return;

        TaskIter task_it = map_it->second;
        int current = task_lane[id];
        if (current >= NUM_PRIORITIES - 1) {
            std::cout << "Task " << id << " already at max priority\n";
            return;
        }

        int new_priority = current + 1;
        task_it->priority = new_priority;

        // O(1) splice: move from current lane to higher lane
        lanes[new_priority].splice(lanes[new_priority].end(),
                                   lanes[current], task_it);
        // Iterator 'task_it' STILL VALID — now in new lane!
        task_lane[id] = new_priority;

        std::cout << "Promoted task " << id << " from priority "
                  << current << " → " << new_priority << "\n";
    }

    Task run_next() {
        // Run highest-priority task
        for (int p = NUM_PRIORITIES - 1; p >= 0; --p) {
            if (!lanes[p].empty()) {
                Task t = lanes[p].front();
                task_map.erase(t.id);
                task_lane.erase(t.id);
                lanes[p].pop_front();
                return t;
            }
        }
        return {-1, "none", -1};
    }

    void print_state() const {
        const char* names[] = {"LOW", "MEDIUM", "HIGH"};
        for (int p = NUM_PRIORITIES - 1; p >= 0; --p) {
            std::cout << "  [" << names[p] << "] ";
            for (const auto& t : lanes[p])
                std::cout << t.id << ":" << t.name << " ";
            std::cout << "\n";
        }
    }
};

int main() {
    Scheduler sched;

    sched.add_task(1, "backup", 0);
    sched.add_task(2, "email", 0);
    sched.add_task(3, "compile", 1);
    sched.add_task(4, "deploy", 1);
    sched.add_task(5, "hotfix", 2);

    std::cout << "\nInitial state:\n";
    sched.print_state();
    // [HIGH]   5:hotfix
    // [MEDIUM] 3:compile 4:deploy
    // [LOW]    1:backup 2:email

    // Promote tasks (O(1) splice!)
    sched.promote(2);   // email: low → medium
    sched.promote(3);   // compile: medium → high

    std::cout << "\nAfter promotions:\n";
    sched.print_state();
    // [HIGH]   5:hotfix 3:compile
    // [MEDIUM] 4:deploy 2:email
    // [LOW]    1:backup

    // Run tasks in priority order
    std::cout << "\nRunning tasks:\n";
    for (int i = 0; i < 5; ++i) {
        Task t = sched.run_next();
        std::cout << "  Ran: " << t.id << ":" << t.name
                  << " (priority " << t.priority << ")\n";
    }
    // Ran: 5:hotfix (priority 2)
    // Ran: 3:compile (priority 2)
    // Ran: 4:deploy (priority 1)
    // Ran: 2:email (priority 1)
    // Ran: 1:backup (priority 0)

    return 0;
}

```

**How it works:**

- Each priority level is a `std::list`. Tasks are promoted by `splice`-ing them from one list to another — O(1), no allocation, no copy.
- A `task_map` stores iterators to each task for O(1) lookup. Since `splice` doesn't invalidate iterators, these handles remain valid after promotion.
- `run_next()` pops the front of the highest non-empty priority queue.
- This pattern is common in operating system schedulers, LRU caches (most-recently-used moves to front), and event systems.

---

## Notes

- **LRU cache:** The classic use case for `splice`. Move accessed items to the front of a list in O(1). Combined with an `unordered_map<Key, list::iterator>`, this gives O(1) access + O(1) LRU eviction.
- **splice within same list:** `a.splice(pos, a, it)` moves element `it` to position `pos` within the same list. O(1).
- **No exception thrown:** `splice` is `noexcept` — it only manipulates pointers.
- **Size update cost:** Range splice between different lists is O(n) in C++11+ because `size()` must be O(1). The actual node relinking is still O(1).
- **`forward_list::splice_after`:** Similar but works with singly-linked lists. You specify the element *before* the splice point.
