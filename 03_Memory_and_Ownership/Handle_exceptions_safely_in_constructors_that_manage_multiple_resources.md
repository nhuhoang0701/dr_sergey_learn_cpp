# Handle Exceptions Safely in Constructors That Manage Multiple Resources

**Category:** Memory & Ownership  
**Item:** #327  
**Reference:** <https://en.cppreference.com/w/cpp/language/constructor>  

---

## Topic Overview

### The Problem: Partial Construction Leaks

When a constructor acquires multiple resources (memory, file handles, locks, etc.), an exception during the second acquisition leaves the first resource leaked. C++ guarantees that **the destructor is NOT called for a partially constructed object**.

### Why Destructors Don't Run on Partial Construction

The C++ standard says: if a constructor throws, the object never fully existed. Only the destructors of **fully constructed subobjects** (base classes and members constructed before the throw) are called.

```cpp

Constructor of MyClass begins:

  1. Base class constructors run          ← these get destroyed if throw happens later
  2. Member initializers run in order     ← already-constructed members get destroyed
  3. Constructor body executes            ← if throw happens here, members ARE destroyed

                                            but ~MyClass() is NOT called

```

### The Fix: RAII Members

If each resource is wrapped in its own RAII object (smart pointer, file handle wrapper, etc.), then partially-constructed members that were fully initialized **will** have their destructors called:

```cpp

class Safe {
    std::unique_ptr<int[]> buf1_;   // RAII member 1
    std::unique_ptr<int[]> buf2_;   // RAII member 2
public:
    Safe()
        : buf1_(new int[1000])      // if this succeeds...
        , buf2_(new int[1000])      // ...and this throws,
    {}                               // buf1_'s destructor runs automatically!
};

```

---

## Self-Assessment

### Q1: Show a constructor that acquires two resources and leaks the first if the second acquisition throws

```cpp

#include <iostream>
#include <stdexcept>
#include <cstring>

// Simulated resource
struct Resource {
    int id;
    Resource(int i) : id(i) {
        std::cout << "  Resource " << id << " acquired\n";
    }
    ~Resource() {
        std::cout << "  Resource " << id << " released\n";
    }
};

// BUGGY: manages two resources with raw pointers
class Leaky {
    Resource* r1_;
    Resource* r2_;
public:
    Leaky(bool second_fails) : r1_(nullptr), r2_(nullptr) {
        r1_ = new Resource(1);       // Acquire first resource

        if (second_fails) {
            throw std::runtime_error("Second resource failed!");
            // r1_ is LEAKED! Destructor won't be called because
            // the object was never fully constructed.
        }

        r2_ = new Resource(2);       // Acquire second resource
    }

    ~Leaky() {
        std::cout << "  ~Leaky() called\n";
        delete r2_;
        delete r1_;
    }
};

int main() {
    std::cout << "=== Case 1: No failure ===\n";
    {
        Leaky ok(false);
    }  // destructor runs, both resources released

    std::cout << "\n=== Case 2: Second resource fails ===\n";
    try {
        Leaky bad(true);  // throws during construction
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
        // ~Leaky() was NEVER called!
        // Resource 1 is LEAKED!
    }

    return 0;
}

```

**Output:**

```text

=== Case 1: No failure ===
  Resource 1 acquired
  Resource 2 acquired
  ~Leaky() called
  Resource 2 released
  Resource 1 released

=== Case 2: Second resource fails ===
  Resource 1 acquired
  Caught: Second resource failed!

```

**Key Issue:** Resource 1 is acquired, but when the constructor throws, `~Leaky()` is never called. Resource 1 leaks.

### Q2: Fix it using RAII wrappers for each resource so the destructor cleans up on partial construction

```cpp

#include <iostream>
#include <memory>
#include <stdexcept>

// Same resource type
struct Resource {
    int id;
    Resource(int i) : id(i) {
        std::cout << "  Resource " << id << " acquired\n";
    }
    ~Resource() {
        std::cout << "  Resource " << id << " released\n";
    }
};

// FIXED: use unique_ptr for each resource
class Safe {
    std::unique_ptr<Resource> r1_;
    std::unique_ptr<Resource> r2_;
public:
    Safe(bool second_fails)
        : r1_(std::make_unique<Resource>(1))   // Acquire first
        , r2_(second_fails ? throw std::runtime_error("Second resource failed!"),
                             std::unique_ptr<Resource>()
                           : std::make_unique<Resource>(2))  // Acquire second
    {
        // Note: the above ternary is unwieldy — usually you'd do:
    }

    ~Safe() {
        std::cout << "  ~Safe() called\n";
    }
};

// Cleaner version using constructor body
class SafeV2 {
    std::unique_ptr<Resource> r1_;
    std::unique_ptr<Resource> r2_;
public:
    SafeV2(bool second_fails) {
        r1_ = std::make_unique<Resource>(1);

        if (second_fails) {
            throw std::runtime_error("Second resource failed!");
            // r1_ is a fully-constructed unique_ptr member, so its
            // destructor WILL be called, releasing Resource 1!
        }

        r2_ = std::make_unique<Resource>(2);
    }

    ~SafeV2() {
        std::cout << "  ~SafeV2() called\n";
    }
};

// Also applicable to non-memory resources via custom deleters
struct FileHandle {
    FILE* fp;
    FileHandle(const char* path) : fp(fopen(path, "w")) {
        if (!fp) throw std::runtime_error("Can't open file");
        std::cout << "  File opened\n";
    }
    ~FileHandle() {
        if (fp) { fclose(fp); std::cout << "  File closed\n"; }
    }
};

class MultiResource {
    std::unique_ptr<Resource> res_;
    FileHandle file_;
public:
    MultiResource(const char* path, bool fail_after_file)
        : res_(std::make_unique<Resource>(1))  // if this succeeds...
        , file_(path)                           // and this succeeds...
    {
        if (fail_after_file) {
            throw std::runtime_error("Late failure");
            // Both res_ and file_ destructors run!
        }
    }
};

int main() {
    std::cout << "=== SafeV2: No failure ===\n";
    {
        SafeV2 ok(false);
    }

    std::cout << "\n=== SafeV2: Second fails ===\n";
    try {
        SafeV2 bad(true);
    } catch (const std::exception& e) {
        std::cout << "  Caught: " << e.what() << "\n";
        // Resource 1 IS properly released because unique_ptr destructor runs!
    }

    return 0;
}

```

**Output:**

```text

=== SafeV2: No failure ===
  Resource 1 acquired
  Resource 2 acquired
  ~SafeV2() called
  Resource 2 released
  Resource 1 released

=== SafeV2: Second fails ===
  Resource 1 acquired
  Caught: Second resource failed!
  Resource 1 released

```

**Key Fix:** When the constructor body throws, all fully-constructed members have their destructors called. `unique_ptr<Resource>` is fully constructed and will destroy Resource 1.

### Q3: Explain why partially constructed objects do not have their destructor called

The C++ standard defines an object's lifetime as beginning when its constructor **completes successfully**. If the constructor throws:

| What happens | Destructor called? |
| --- | --- |
| Base class subobjects (already constructed) | **Yes** — in reverse order |
| Member subobjects (already constructed) | **Yes** — in reverse declaration order |
| The object itself (~MyClass) | **No** — it never existed |
| Array elements (partially constructed array) | **Yes** for completed elements |

```cpp

#include <iostream>
#include <stdexcept>

struct Base {
    Base()  { std::cout << "  Base()\n"; }
    ~Base() { std::cout << "  ~Base()\n"; }
};

struct MemberA {
    MemberA()  { std::cout << "  MemberA()\n"; }
    ~MemberA() { std::cout << "  ~MemberA()\n"; }
};

struct MemberB {
    MemberB()  { std::cout << "  MemberB()\n"; }
    ~MemberB() { std::cout << "  ~MemberB()\n"; }
};

struct MemberC {
    MemberC()  { std::cout << "  MemberC() - THROWS!\n"; throw std::runtime_error("fail"); }
    ~MemberC() { std::cout << "  ~MemberC()\n"; }
};

class MyClass : public Base {
    MemberA a_;
    MemberB b_;
    MemberC c_;   // throws during construction
public:
    MyClass() {
        std::cout << "  MyClass body (never reached)\n";
    }
    ~MyClass() {
        std::cout << "  ~MyClass() (never called)\n";
    }
};

int main() {
    std::cout << "=== Construction attempt ===\n";
    try {
        MyClass obj;
    } catch (...) {
        std::cout << "  Exception caught\n";
    }

    // What we observe:
    // 1. Base() constructed ✓
    // 2. MemberA() constructed ✓
    // 3. MemberB() constructed ✓
    // 4. MemberC() throws ✗
    // 5. ~MemberB() called (reverse order) ✓
    // 6. ~MemberA() called ✓
    // 7. ~Base() called ✓
    // 8. ~MyClass() NOT called (object never fully existed)

    return 0;
}

```

**Output:**

```text

=== Construction attempt ===
  Base()
  MemberA()
  MemberB()
  MemberC() - THROWS!
  ~MemberB()
  ~MemberA()
  ~Base()
  Exception caught

```

**Rules:**

1. **The object's destructor is never called** because the object's lifetime never began.
2. **Already-constructed subobjects ARE destroyed** in reverse order of construction.
3. This is why **each resource must be its own RAII wrapper** — the wrapper's destructor WILL run even during partial construction.

---

## Notes

- The function-try-block (`try`/`catch` around the entire constructor) can catch exceptions from member initializers, but it must rethrow — you cannot "fix" a failed construction.
- `std::make_unique` and `std::make_shared` are single-allocation calls, so they don't have the "first alloc succeeds, second throws" problem within themselves.
- In member initializer lists, construction order follows **declaration order in the class**, not the order in the initializer list. Be careful of dependencies between members.
