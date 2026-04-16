# Know the initialization rules for static local variables and their thread-safety

**Category:** Core Language Fundamentals  
**Standard:** C++11  
**Reference:** <https://en.cppreference.com/w/cpp/language/storage_duration#Static_local_variables>  

---

## Topic Overview

### Static Local Variable Initialization Rules

A **function-local static variable** is declared with `static` inside a function. Key rules:

1. **Initialized exactly once** — the first time execution passes through the declaration.
2. **Thread-safe initialization** (C++11+) — if multiple threads call the function simultaneously, only one thread performs initialization; others block until it's done.
3. **Lifetime** — from first initialization until program exit.
4. **Destruction** — at program exit, in **reverse order** of construction (registered with `atexit`).

```cpp

Logger& get_logger() {
    static Logger instance("app.log");  // Created once, first time called
    return instance;                     // Thread-safe in C++11+
}

```

### How The Compiler Implements Thread Safety

The compiler emits a hidden guard variable:

```cpp

// Conceptual compiler output for the above:
Logger& get_logger() {
    static bool __guard = false;  // Hidden guard variable
    alignas(Logger) static char __storage[sizeof(Logger)];

    if (!__guard) {                    // Fast path: already initialized
        // Acquire lock (implementation-specific)
        if (!__guard) {                // Double-check under lock
            new (__storage) Logger("app.log");
            __guard = true;
            // Register destructor with atexit
        }
        // Release lock
    }
    return *reinterpret_cast<Logger*>(__storage);
}

```

### Why This Obsoletes Double-Checked Locking

Before C++11, the common pattern for thread-safe singletons was "double-checked locking":

```cpp

// Pre-C++11 BROKEN pattern:
Singleton* instance = nullptr;
std::mutex mtx;

Singleton* get_instance() {
    if (!instance) {                    // First check (no lock — may see partial write!)
        std::lock_guard lock(mtx);
        if (!instance) {                // Second check (under lock)
            instance = new Singleton(); // BUG: reorder! Pointer assigned before construction
        }
    }
    return instance;
}

```

**This is broken** because without memory barriers, the compiler/CPU may assign to `instance` before the `Singleton` constructor finishes. Another thread sees a non-null pointer but reads an unconstructed object.

C++11 fixed this by making `static` local initialization inherently thread-safe.

---

## Self-Assessment

### Q1: Show that a function-local static is initialized exactly once, even with concurrent callers

```cpp

#include <iostream>
#include <thread>
#include <vector>
#include <atomic>

struct ExpensiveResource {
    int id;
    static std::atomic<int> construction_count;

    ExpensiveResource() : id(++construction_count) {
        std::cout << "Constructing resource #" << id
                  << " on thread " << std::this_thread::get_id() << "\n";
        // Simulate expensive initialization
        std::this_thread::sleep_for(std::chrono::milliseconds(100));
    }

    ~ExpensiveResource() {
        std::cout << "Destroying resource #" << id << "\n";
    }
};

std::atomic<int> ExpensiveResource::construction_count{0};

ExpensiveResource& get_resource() {
    static ExpensiveResource instance;  // Initialized ONCE, thread-safe
    return instance;
}

int main() {
    std::vector<std::thread> threads;

    // Launch 10 threads that all try to get the resource simultaneously
    for (int i = 0; i < 10; ++i) {
        threads.emplace_back([i]() {
            auto& res = get_resource();
            std::cout << "Thread " << i << " got resource #" << res.id << "\n";
        });
    }

    for (auto& t : threads) t.join();

    std::cout << "Total constructions: "
              << ExpensiveResource::construction_count << "\n";
    // Output: Total constructions: 1
    // Only ONE construction, despite 10 concurrent callers!

    return 0;
}

```

**Output (order may vary):**

```text

Constructing resource #1 on thread 140234567890
Thread 0 got resource #1
Thread 3 got resource #1
Thread 1 got resource #1
...
Total constructions: 1

```

**How this works:**

- The first thread to reach `static ExpensiveResource instance;` starts construction.
- All other threads **block** (wait) until construction completes.
- After initialization, all subsequent calls skip construction entirely (fast path).
- The standard guarantees: **"If control enters the declaration concurrently while the variable is being initialized, the concurrent execution shall wait for completion of the initialization."** [stmt.dcl]

### Q2: The double-checked locking problem and why C++11 obsoletes it

**The double-checked locking problem (DCLP):**

```cpp

#include <mutex>
#include <memory>

class Singleton {
    static Singleton* instance;
    static std::mutex mtx;

public:
    // BROKEN double-checked locking pattern:
    static Singleton* get_instance_BROKEN() {
        if (instance == nullptr) {          // 1st check: no lock
            std::lock_guard<std::mutex> lock(mtx);
            if (instance == nullptr) {      // 2nd check: under lock
                instance = new Singleton(); // PROBLEM!
                // The compiler/CPU may reorder this as:
                // 1. Allocate memory
                // 2. Assign pointer to instance  ← another thread sees non-null
                // 3. Call constructor             ← but object isn't constructed yet!
            }
        }
        return instance;  // May return pointer to unconstructed object
    }

    // CORRECT C++11 fix using function-local static:
    static Singleton& get_instance() {
        static Singleton instance;  // Thread-safe, no manual locking needed
        return instance;
    }
};

```

**Why DCLP fails without C++11 atomics:**

1. `instance = new Singleton()` involves three steps: allocate, construct, assign.
2. The CPU may **reorder** assign before construct (out-of-order execution).
3. Another thread executing the first `if` check sees `instance != nullptr` and skips the lock.
4. It then uses a pointer to **uninitialized memory** → UB.

**Why C++11 static locals fix this permanently:**

- The compiler inserts the proper memory barriers and guard variables.
- No user-written locking or atomic operations needed.
- The code is simpler, correct, and often faster (compiler-optimized guard check).

### Q3: Demonstrate that destructors of static locals are called at exit in reverse order

```cpp

#include <iostream>

struct Tracer {
    std::string name;
    Tracer(std::string n) : name(std::move(n)) {
        std::cout << "  Construct: " << name << "\n";
    }
    ~Tracer() {
        std::cout << "  Destroy:   " << name << "\n";
    }
};

Tracer& get_first() {
    static Tracer instance("First");
    return instance;
}

Tracer& get_second() {
    static Tracer instance("Second");
    return instance;
}

Tracer& get_third() {
    static Tracer instance("Third");
    return instance;
}

int main() {
    std::cout << "=== Initialization order ===\n";
    get_first();     // Constructs "First"
    get_second();    // Constructs "Second"
    get_third();     // Constructs "Third"

    std::cout << "\n=== main() ending ===\n";
    // Destructors called in REVERSE order of construction:
    return 0;
}

```

**Output:**

```text

=== Initialization order ===
  Construct: First
  Construct: Second
  Construct: Third

=== main() ending ===
  Destroy:   Third
  Destroy:   Second
  Destroy:   First

```

**How this works:**

- Each static local is registered with `atexit()` (or equivalent) when constructed.
- `atexit` handlers run in LIFO (last-in-first-out) order at program exit.
- **Danger:** If "Third" depends on "First" during its destruction, and "First" is already destroyed → UB.
- This is the **static destruction order fiasco** — the mirror image of the initialization order fiasco.

**Mitigation:**

```cpp

// "Leaky singleton" — intentionally never destroyed to avoid destruction order issues
Tracer& get_immortal() {
    static Tracer* instance = new Tracer("Immortal");
    return *instance;  // Leaked — but never destroyed, so no dangling references
}

```

---

## Notes

- C++11 guarantees thread-safe initialization of function-local statics — use this instead of manual locking.
- Double-checked locking is obsoleted by `static` locals. Don't write DCLP in modern C++.
- Static locals are destroyed in reverse construction order at program exit.
- Beware the destruction order fiasco — if globals/statics depend on each other during cleanup.
- Compiler flag `-fno-threadsafe-statics` (GCC/Clang) disables the thread-safety overhead for embedded/single-threaded code.

```cpp

// Your practice code

```
