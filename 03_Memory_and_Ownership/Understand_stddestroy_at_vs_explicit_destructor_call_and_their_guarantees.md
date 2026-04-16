# Understand std::destroy_at vs Explicit Destructor Call and Their Guarantees

**Category:** Memory & Ownership  
**Item:** #290  
**Reference:** <https://en.cppreference.com/w/cpp/memory/destroy_at>  

---

## Topic Overview

### The Low-Level Object Lifetime Functions (C++17/20)

| Function | Since | Purpose |
| --- | --- | --- |
| `std::destroy_at(p)` | C++17 | Destroy object at `p` |
| `std::destroy(first, last)` | C++17 | Destroy range |
| `std::destroy_n(first, n)` | C++17 | Destroy `n` objects |
| `std::construct_at(p, args...)` | C++20 | Construct object at `p` |

### `std::destroy_at` vs `p->~T()`

| Aspect | `p->~T()` | `std::destroy_at(p)` |
| --- | --- | --- |
| **Arrays** | Destroys only one element | Destroys entire array (C++20) |
| **Constexpr** | No | Yes (C++20) |
| **Readability** | Template-dependent syntax issues | Clean, generic |
| **Type safety** | Must spell the exact type | Deduces type from pointer |

### Why `destroy_at` is Safer for Arrays

```cpp

// T arr[5];

// Explicit: ONLY destroys item at arr
// (doesn't destroy all 5 elements!)
arr->~T();

// destroy_at on array pointer: destroys ALL elements (C++20)
std::destroy_at(&arr);  // calls ~T() for each element

```

---

## Self-Assessment

### Q1: Use `std::destroy_at` instead of `p->~T()` and explain why `destroy_at` is safer for arrays

```cpp

#include <iostream>
#include <memory>
#include <new>
#include <cstring>

struct Widget {
    std::string name;
    Widget(const char* n) : name(n) {
        std::cout << "  Constructed: " << name << "\n";
    }
    ~Widget() {
        std::cout << "  Destroyed: " << name << "\n";
    }
};

int main() {
    std::cout << "=== 1. Single object: explicit destructor ===\n";
    {
        alignas(Widget) unsigned char buf[sizeof(Widget)];
        Widget* p = new (buf) Widget("Explicit");
        p->~Widget();  // Works, but verbose
    }

    std::cout << "\n=== 2. Single object: std::destroy_at ===\n";
    {
        alignas(Widget) unsigned char buf[sizeof(Widget)];
        Widget* p = new (buf) Widget("DestroyAt");
        std::destroy_at(p);  // Cleaner, deduces type
    }

    std::cout << "\n=== 3. Array: explicit destructor (WRONG) ===\n";
    {
        alignas(Widget) unsigned char buf[sizeof(Widget) * 3];
        Widget* arr = reinterpret_cast<Widget*>(buf);
        new (&arr[0]) Widget("A");
        new (&arr[1]) Widget("B");
        new (&arr[2]) Widget("C");

        // p->~Widget() only destroys ONE element!
        // You'd need a manual loop:
        for (int i = 2; i >= 0; --i)
            arr[i].~Widget();
    }

    std::cout << "\n=== 4. Range: std::destroy (correct, generic) ===\n";
    {
        alignas(Widget) unsigned char buf[sizeof(Widget) * 3];
        Widget* arr = reinterpret_cast<Widget*>(buf);
        new (&arr[0]) Widget("X");
        new (&arr[1]) Widget("Y");
        new (&arr[2]) Widget("Z");

        // Destroys all elements in range — generic, safe
        std::destroy(arr, arr + 3);
    }

    std::cout << "\n=== 5. Why destroy_at is better in templates ===\n";
    // In templates, p->~T() can have parsing issues:
    //   p->~T()       — might confuse parser with dependent names
    //   destroy_at(p) — always works, no disambiguation needed

    return 0;
}
// Expected output:
// === 1. Single object: explicit destructor ===
//   Constructed: Explicit
//   Destroyed: Explicit
//
// === 2. Single object: std::destroy_at ===
//   Constructed: DestroyAt
//   Destroyed: DestroyAt
//
// === 3. Array: explicit destructor (WRONG) ===
//   Constructed: A
//   Constructed: B
//   Constructed: C
//   Destroyed: C
//   Destroyed: B
//   Destroyed: A
//
// === 4. Range: std::destroy (correct, generic) ===
//   Constructed: X
//   Constructed: Y
//   Constructed: Z
//   Destroyed: X
//   Destroyed: Y
//   Destroyed: Z

```

### Q2: Show that calling the destructor directly twice is undefined behavior

```cpp

#include <iostream>
#include <string>
#include <new>

struct Tracked {
    std::string data;
    Tracked(const char* s) : data(s) {
        std::cout << "Constructed: " << data << "\n";
    }
    ~Tracked() {
        std::cout << "Destroyed: data='" << data << "'\n";
    }
};

int main() {
    std::cout << "=== Double destructor is UB ===\n\n";

    // Using raw storage to avoid automatic destructor call
    alignas(Tracked) unsigned char buf[sizeof(Tracked)];
    Tracked* p = new (buf) Tracked("Important");

    // First destruction: OK
    std::destroy_at(p);

    // Second destruction: UNDEFINED BEHAVIOR!
    // std::destroy_at(p);  // UB: object lifetime already ended
    // p->~Tracked();       // Also UB

    // What can happen with double destruction:
    // 1. std::string's destructor frees internal buffer twice → heap corruption
    // 2. Double-free detected by allocator → crash
    // 3. Appears to "work" with trivial types (still technically UB)
    // 4. Memory corruption that manifests much later

    std::cout << "\n=== Safe pattern: use a flag ===\n";

    struct SafeSlot {
        alignas(Tracked) unsigned char storage[sizeof(Tracked)];
        bool alive = false;

        Tracked* ptr() { return reinterpret_cast<Tracked*>(storage); }

        void construct(const char* s) {
            if (alive) destroy();  // Prevent double-construct leak
            new (storage) Tracked(s);
            alive = true;
        }
        void destroy() {
            if (alive) {
                std::destroy_at(ptr());
                alive = false;     // Prevents double-destroy
            }
        }
    };

    SafeSlot slot;
    slot.construct("Safely Managed");
    slot.destroy();   // OK: destroys
    slot.destroy();   // OK: no-op (alive == false)

    std::cout << "Double destroy safely avoided.\n";

    return 0;
}
// Expected output:
// === Double destructor is UB ===
//
// Constructed: Important
// Destroyed: data='Important'
//
// === Safe pattern: use a flag ===
// Constructed: Safely Managed
// Destroyed: data='Safely Managed'
// Double destroy safely avoided.

```

### Q3: Implement a manually managed object pool using `construct_at` and `destroy_at`

```cpp

#include <iostream>
#include <memory>
#include <new>
#include <bitset>
#include <cstddef>
#include <cassert>

template<typename T, size_t Capacity>
class ObjectPool {
    // Raw storage — no constructors called
    alignas(T) unsigned char storage_[Capacity * sizeof(T)];
    std::bitset<Capacity> in_use_;

    T* slot(size_t i) {
        return reinterpret_cast<T*>(storage_ + i * sizeof(T));
    }

public:
    ObjectPool() = default;

    ~ObjectPool() {
        // Destroy all live objects
        for (size_t i = 0; i < Capacity; ++i) {
            if (in_use_[i]) {
                std::destroy_at(slot(i));
            }
        }
    }

    // Non-copyable, non-movable
    ObjectPool(const ObjectPool&) = delete;
    ObjectPool& operator=(const ObjectPool&) = delete;

    template<typename... Args>
    T* construct(Args&&... args) {
        for (size_t i = 0; i < Capacity; ++i) {
            if (!in_use_[i]) {
                T* p = std::construct_at(slot(i), std::forward<Args>(args)...);
                in_use_.set(i);
                return p;
            }
        }
        return nullptr;  // Pool exhausted
    }

    void destroy(T* ptr) {
        // Find which slot this pointer belongs to
        auto offset = reinterpret_cast<unsigned char*>(ptr) - storage_;
        size_t index = static_cast<size_t>(offset) / sizeof(T);

        assert(index < Capacity && "Pointer not from this pool");
        assert(in_use_[index] && "Slot already free");

        std::destroy_at(ptr);
        in_use_.reset(index);
    }

    size_t active_count() const { return in_use_.count(); }
    size_t capacity() const { return Capacity; }
};

struct Actor {
    std::string name;
    int hp;

    Actor(std::string n, int h) : name(std::move(n)), hp(h) {
        std::cout << "  [+] Actor '" << name << "' (HP:" << hp << ")\n";
    }
    ~Actor() {
        std::cout << "  [-] Actor '" << name << "' destroyed\n";
    }
};

int main() {
    std::cout << "=== Object Pool with construct_at / destroy_at ===\n\n";

    ObjectPool<Actor, 4> pool;

    Actor* a = pool.construct("Warrior", 100);
    Actor* b = pool.construct("Mage", 60);
    Actor* c = pool.construct("Archer", 80);

    std::cout << "\nActive: " << pool.active_count()
              << "/" << pool.capacity() << "\n\n";

    // Destroy one — slot is recycled
    pool.destroy(b);
    std::cout << "\nAfter destroying Mage: "
              << pool.active_count() << "/" << pool.capacity() << "\n\n";

    // Reuse the freed slot
    Actor* d = pool.construct("Healer", 50);
    Actor* e = pool.construct("Thief", 70);

    std::cout << "\nActive: " << pool.active_count()
              << "/" << pool.capacity() << "\n";

    // Pool full — returns nullptr
    Actor* f = pool.construct("Ghost", 1);
    std::cout << "Allocate when full: " << (f ? "success" : "nullptr") << "\n";

    std::cout << "\n--- Pool destructor cleans up remaining ---\n";
    // Pool destructor destroys a, d, e, c automatically
    return 0;
}
// Expected output:
// === Object Pool with construct_at / destroy_at ===
//
//   [+] Actor 'Warrior' (HP:100)
//   [+] Actor 'Mage' (HP:60)
//   [+] Actor 'Archer' (HP:80)
//
// Active: 3/4
//
//   [-] Actor 'Mage' destroyed
//
// After destroying Mage: 2/4
//
//   [+] Actor 'Healer' (HP:50)
//   [+] Actor 'Thief' (HP:70)
//
// Active: 4/4
// Allocate when full: nullptr
//
// --- Pool destructor cleans up remaining ---
//   [-] Actor 'Warrior' destroyed
//   [-] Actor 'Healer' destroyed
//   [-] Actor 'Archer' destroyed
//   [-] Actor 'Thief' destroyed

```

---

## Notes

- `std::destroy_at` is the canonical way to end an object's lifetime in uninitialized memory.
- `std::construct_at` (C++20) replaces placement new in `constexpr` contexts.
- Calling a destructor twice on the same object is always UB, even for trivially destructible types (in the standard's formal model).
- Use `std::destroy(first, last)` to destroy a range — it handles each element and is exception-safe.
- Object pools eliminate per-object allocation overhead: one big allocation, many construct/destroy cycles.
