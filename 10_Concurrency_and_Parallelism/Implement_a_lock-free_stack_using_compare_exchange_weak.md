# Implement a lock-free stack using compare_exchange_weak

**Category:** Concurrency & Parallelism  
**Item:** #371  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/atomic/atomic/compare_exchange>  

---

## Topic Overview

A **lock-free stack** (Treiber stack) uses `std::atomic<Node*>` and `compare_exchange_weak` (CAS) instead of mutexes. Multiple threads can push and pop concurrently without ever blocking — if one thread is delayed, others still make progress.

### Treiber Stack Algorithm

```cpp

Push(value):                          Pop():
  new_node->next = head.load()         old_head = head.load()
  while (!head.CAS(new_node->next,     while (old_head &&
              new_node))                    !head.CAS(old_head,
    ; // retry (another thread won)              old_head->next))
                                            ; // retry
                                        return old_head->data

CAS = compare_exchange_weak:
  "If head still equals expected, set head = desired and return true.
   Otherwise, update expected to current head and return false."

```

### Visual: Concurrent Push

```cpp

Thread 1: push(A)           Thread 2: push(B)
                            
head → [C] → [D] → null    head → [C] → [D] → null
A.next = C                  B.next = C
CAS(head, C→A) ✓ success   CAS(head, C→B) ✗ fail! (head is now A)
head → [A] → [C] → [D]     B.next = A  (updated by CAS)
                            CAS(head, A→B) ✓ success
                            head → [B] → [A] → [C] → [D]

```

---

## Self-Assessment

### Q1: Build a Treiber stack using atomic<Node*> and compare_exchange_weak for push and pop

**Answer:**

```cpp

#include <atomic>
#include <iostream>
#include <thread>
#include <vector>
#include <optional>

template<typename T>
class LockFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    std::atomic<Node*> head_{nullptr};

public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        new_node->next = head_.load(std::memory_order_relaxed);

        // CAS loop: try to set head to new_node
        // If head changed since we read it, CAS updates new_node->next
        // to the current head and we retry
        while (!head_.compare_exchange_weak(
            new_node->next,  // expected (updated on failure)
            new_node,        // desired
            std::memory_order_release,
            std::memory_order_relaxed)) {
            // Retry — another thread pushed/popped between load and CAS
        }
    }

    std::optional<T> pop() {
        Node* old_head = head_.load(std::memory_order_relaxed);

        while (old_head) {
            // Try to advance head to old_head->next
            if (head_.compare_exchange_weak(
                old_head,           // expected
                old_head->next,     // desired
                std::memory_order_acquire,
                std::memory_order_relaxed)) {

                T value = std::move(old_head->data);
                delete old_head; // NOTE: simplified — has ABA issues (see Q3)
                return value;
            }
            // CAS failed — old_head updated to current head, retry
        }
        return std::nullopt; // stack is empty
    }

    ~LockFreeStack() {
        while (pop()) {} // drain remaining nodes
    }
};

int main() {
    LockFreeStack<int> stack;

    // Push from multiple threads
    std::vector<std::thread> pushers;
    for (int t = 0; t < 4; ++t) {
        pushers.emplace_back([&, t] {
            for (int i = 0; i < 1000; ++i)
                stack.push(t * 1000 + i);
        });
    }
    for (auto& t : pushers) t.join();

    // Pop all — should get exactly 4000 items
    int count = 0;
    while (stack.pop()) ++count;
    std::cout << "Popped: " << count << " items\n";
    // Output: Popped: 4000 items
}

```

**Explanation:** The push operation creates a new node, sets its `next` to the current head, then attempts a CAS to swing the head pointer. If another thread modified head between our load and CAS, the CAS fails, updates `new_node->next` to the actual current head, and we retry. Pop works similarly — we try to advance head to `head->next`.

### Q2: Explain why compare_exchange_weak is preferred over strong in a retry loop

**Answer:**

```cpp

#include <atomic>
#include <iostream>

int main() {
    std::atomic<int> x{42};
    int expected = 42;

    // compare_exchange_WEAK:
    // - May SPURIOUSLY fail even if x == expected
    // - Returns false without performing the exchange
    // - On some architectures (ARM, POWER), this maps to LL/SC instructions
    //   which can fail due to cache contention, not actual value change
    //
    // compare_exchange_STRONG:
    // - Only fails if x != expected (no spurious failures)
    // - Internally, it's a LOOP around the weak version:
    //   while (!weak(expected, desired)) {
    //       if (expected != original_expected) return false; // genuine fail
    //   }
    //   return true;

    // IN A RETRY LOOP (like Treiber stack push/pop):
    //
    //   while (!x.compare_exchange_weak(expected, desired)) { }
    //
    // We're ALREADY in a loop, so spurious failures just cause an extra iteration.
    // Using "strong" would add an INNER loop (to eliminate spurious failures)
    // inside our OUTER loop — redundant work!

    // WEAK in a loop (optimal):
    while (!x.compare_exchange_weak(expected, 99)) {
        // Retry — expected is auto-updated
    }
    std::cout << "x = " << x.load() << "\n"; // 99

    // STRONG without a loop (when you need exactly one attempt):
    x.store(42);
    expected = 42;
    bool success = x.compare_exchange_strong(expected, 100);
    std::cout << "Strong success: " << std::boolalpha << success << "\n"; // true

    // SUMMARY:
    // Use WEAK when: you have a retry loop (CAS loops, lock-free algorithms)
    // Use STRONG when: you need a single attempt with a definitive result
    //                  (e.g., conditional update without retry)

    // PERFORMANCE on ARM/POWER:
    //   weak  → single LL/SC pair → fast
    //   strong → LL/SC loop until success or genuine failure → slower per call
    // On x86: both compile to the same CMPXCHG instruction (no difference)
}

```

**Explanation:** `compare_exchange_weak` can spuriously fail on LL/SC architectures (ARM, POWER) because the store-conditional can fail due to cache-line contention, interrupts, or other cores' activity — even if the value hasn't changed. Since lock-free algorithms already loop, the spurious failure just costs one extra iteration. Using `strong` would add an unnecessary inner loop to suppress spurious failures, wasting cycles.

### Q3: Show the ABA risk in the Treiber stack and how to mitigate it

**Answer:**

```cpp

#include <atomic>
#include <iostream>
#include <cstdint>

// === THE ABA PROBLEM ===
//
// Thread 1 (pop):                Thread 2 (concurrent):
// 1. Read head → A               
// 2. Read A->next → B            
//    (About to CAS head from A to B)
//    ...gets preempted...         
//                                 3. Pop A (head = B)
//                                 4. Pop B (head = C)
//                                 5. Push A back (recycle) → head = A → C
//                                    (A is now reused but A->next ≠ B!)
// 6. CAS(head, A → B) succeeds!  
//    Because head IS A again — but A->next is now C, not B!
//    head = B (DANGLING!) — stack is corrupted
//
// The value went A → B → A. CAS sees A and thinks nothing changed.
// But the MEANING of A has changed — this is the "ABA" problem.

// === MITIGATION: Tagged pointer (version counter) ===
template<typename T>
class ABAFreeStack {
    struct Node {
        T data;
        Node* next;
        Node(T val) : data(std::move(val)), next(nullptr) {}
    };

    // Pack a pointer + version counter into a single atomic
    struct TaggedPtr {
        Node* ptr;
        uintptr_t tag; // version counter

        bool operator==(const TaggedPtr& o) const {
            return ptr == o.ptr && tag == o.tag;
        }
    };

    // NOTE: requires std::atomic<TaggedPtr> to be lock-free
    // On x86-64, this works with 128-bit CAS (CMPXCHG16B)
    std::atomic<TaggedPtr> head_{{nullptr, 0}};

public:
    void push(T value) {
        Node* new_node = new Node(std::move(value));
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);

        TaggedPtr new_head;
        new_head.ptr = new_node;

        do {
            new_node->next = old_head.ptr;
            new_head.tag = old_head.tag + 1; // increment version!
        } while (!head_.compare_exchange_weak(
            old_head, new_head,
            std::memory_order_release,
            std::memory_order_relaxed));
    }

    std::optional<T> pop() {
        TaggedPtr old_head = head_.load(std::memory_order_relaxed);

        while (old_head.ptr) {
            TaggedPtr new_head{old_head.ptr->next, old_head.tag + 1};

            if (head_.compare_exchange_weak(
                old_head, new_head,
                std::memory_order_acquire,
                std::memory_order_relaxed)) {

                T value = std::move(old_head.ptr->data);
                // Don't delete immediately — use hazard pointers or
                // epoch-based reclamation for safe memory reclamation
                delete old_head.ptr; // simplified
                return value;
            }
        }
        return std::nullopt;
    }
};

int main() {
    // The tag (version counter) prevents ABA:
    // Even if the pointer returns to the same address A,
    // the tag has changed from 0 to 2, so CAS fails correctly.
    //
    // pop reads: {A, tag=0}
    // Meanwhile: pop A (tag→1), pop B (tag→2), push A back (tag→3)
    // CAS compares {A, 0} vs {A, 3} → NOT EQUAL → fails → retry ✓

    std::cout << "ABA-safe stack operational\n";

    // OTHER ABA MITIGATIONS:
    // 1. Hazard pointers (C++26: std::hazard_pointer)
    //    - Each thread publishes pointers it's reading
    //    - Deletions deferred until no hazard pointer references the node
    //
    // 2. Epoch-based reclamation (RCU-like)
    //    - Readers enter/exit epochs
    //    - Deleted nodes reclaimed when no reader is in the old epoch
    //
    // 3. Don't reuse node memory (allocate fresh, use GC)
    //    - Wasteful but eliminates ABA by design
}

```

**Explanation:** The ABA problem occurs when a CAS succeeds because the value matches, but the meaning has changed. A node pointer returns to the same address after being popped and re-pushed, so CAS can't detect the intervening changes. The tagged-pointer approach adds a monotonically incrementing version counter — even if the pointer is the same, the tag differs, causing CAS to correctly fail and retry.

---

## Notes

- **Memory ordering:** Push uses `release` (publishes the new node), pop uses `acquire` (reads the node's data). This ensures the node's contents are visible after pop.
- **Memory reclamation:** The simplified `delete` in pop has a use-after-free risk if another thread is reading the same node. Use hazard pointers, epoch-based reclamation, or `std::hazard_pointer` (C++26).
- **128-bit CAS:** The tagged-pointer approach requires double-width CAS (`CMPXCHG16B` on x86-64). Compile with `-mcx16` on GCC/Clang.
- **`compare_exchange_weak` loop pattern:** Always update `expected` (the first parameter is in/out) — no manual reload needed.
- Test with `-fsanitize=thread` and stress testing to verify lock-free correctness.
