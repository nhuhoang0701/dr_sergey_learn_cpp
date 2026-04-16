# Know null pointer dereference UB nuances: passing vs dereferencing

**Category:** Undefined Behavior Deep Dive  
**Standard:** C++17  
**Reference:** <https://en.cppreference.com/w/cpp/language/ub>  

---

## Topic Overview

### The Basic Rule

Dereferencing a null pointer is undefined behavior in C++:

```cpp

int* p = nullptr;
int x = *p;   // UB: dereferencing null pointer
p->foo();      // UB: member access through null pointer

```

However, the nuances around null pointers are more subtle than many developers realize.

### Forming a Reference to Null — UB Even Without Access

```cpp

int* p = nullptr;
int& r = *p;    // UB! Even if you never read through r.
                 // The act of binding a reference to a dereferenced
                 // null pointer is itself UB.

```

The standard says a reference must be bound to a valid object. Since `*p` where `p == nullptr` does not denote a valid object, the binding is UB regardless of subsequent use.

### Passing Null Pointers — Not UB by Itself

Simply **having** a null pointer, passing it around, or comparing it is perfectly fine:

```cpp

void process(int* p) {
    if (p == nullptr) return;  // Fine — comparing null is valid
    *p = 42;                   // Fine — p is not null here
}

int* p = nullptr;
int* q = p;           // Fine — copying a null pointer
bool b = (p == q);    // Fine — comparing null pointers
process(p);           // Fine — passing null to a function

```

### When the Compiler Exploits Null Dereference UB

Since dereferencing null is UB, the compiler can assume any dereferenced pointer was **not null**:

```cpp

void dangerous(int* p) {
    int x = *p;          // Dereference — compiler assumes p != nullptr
    if (p == nullptr) {  // Compiler: "I already know p != nullptr"
        handle_error();  // THIS CODE IS REMOVED by the optimizer!
    }
    use(x);
}

```

This is a real optimization that GCC and Clang perform at `-O2` and above. The null check **after** the dereference is eliminated because the compiler assumes the program has no UB, therefore `p` cannot be null.

### The `this` Pointer Is Never Null

The standard guarantees `this` is never null inside a member function called on a valid object. Compilers exploit this:

```cpp

struct Widget {
    int value;
    
    int get_value() {
        if (this == nullptr) return -1;  // REMOVED by optimizer!
        return value;
    }
};

Widget* w = nullptr;
w->get_value();  // UB: calling member function on null pointer
                 // The compiler may not even generate the null check

```

Even though some legacy codebases check `this == nullptr` as a "safety" measure, this is UB-based code that modern optimizers will break.

### `delete nullptr` Is Defined

One explicit exception: deleting a null pointer is a no-op, **not** UB:

```cpp

int* p = nullptr;
delete p;      // Defined: does nothing
delete[] p;    // Defined: does nothing

```

### Null Pointer Arithmetic — UB

```cpp

int* p = nullptr;
int* q = p + 0;   // UB in C++, even though offset is 0!
                   // (The standard requires the pointer operand
                   //  to point to an array element)

```

Note: C allows `p + 0` on null pointers. C++ does not. This is a difference that bites interop code.

### Member Access on Null — Always UB

```cpp

struct S {
    int x;
    static int y;
    void f();
    static void g();
};

S* p = nullptr;
p->x;       // UB: accessing non-static member through null
p->f();     // UB: calling non-static member function on null
p->y;       // Technically UB (member access through null), but
            // compilers typically handle it since y doesn't use 'this'
S::g();     // Fine: static call, no object needed
p->g();     // UB per standard, but equivalent to S::g() in practice

```

### Practical Patterns to Avoid Null Dereference

```cpp

// 1. Use references instead of pointers when null is not valid
void process(const Widget& w);  // Cannot be null

// 2. Use std::optional for "maybe no value"
std::optional<Widget> find_widget(int id);

// 3. Use gsl::not_null<T*> to document and enforce non-null
#include <gsl/gsl>
void process(gsl::not_null<Widget*> w);  // Checked at construction

// 4. Use assert for internal invariants
void internal_process(Widget* w) {
    assert(w != nullptr && "internal_process requires non-null widget");
    w->do_work();
}

// 5. Always check BEFORE dereferencing, never after
void safe(int* p) {
    if (!p) return;   // Check FIRST
    int x = *p;       // Then dereference
    use(x);
}

```

---

## Self-Assessment

### Q1: Why does the compiler remove `if (p == nullptr)` after `*p`

The compiler reasons: "The program dereferences `p` on the line above. If `p` were null, that would be UB, and the standard allows me to assume no UB exists in a valid program. Therefore, `p` is not null at this point, and the null check is dead code."

This is correct per the standard but can remove **intended** safety checks if the programmer wrote them in the wrong order. The fix: always check for null **before** the first dereference.

### Q2: Is `static_cast<int*>(nullptr) + 0` defined

No. In C++, pointer arithmetic requires the pointer to point to an element of an array (or one past the end). A null pointer does not point to any object, so `nullptr + 0` is UB in C++.

In C (C11+), `nullptr + 0` is explicitly permitted and yields `nullptr`. This difference can cause bugs in code shared between C and C++.

### Q3: Show the correct order for null-check and dereference

```cpp

// WRONG — null check after dereference (check may be optimized away)
void wrong(int* p) {
    int x = *p;              // Compiler assumes p != nullptr here
    if (p == nullptr) {      // Dead code — removed!
        return;
    }
    use(x);
}

// CORRECT — null check before dereference
void correct(int* p) {
    if (p == nullptr) {      // Check first
        return;
    }
    int x = *p;              // Safe — p is definitely not null
    use(x);
}

// BEST — use references to make null impossible
void best(int& x) {         // Cannot be null by construction
    use(x);
}

```

---

## Notes

- Use `-fno-delete-null-pointer-checks` (GCC/Clang) to prevent null check elimination — useful for kernel code where null dereferences are recoverable
- Linux kernel uses this flag because page 0 can be mapped and handled by fault handlers
- `std::launder` does not make null pointer dereference valid
- Static analyzers (Clang-Tidy `bugprone-null-dereference`, Coverity) catch many null dereference bugs at compile time
- `[[gnu::nonnull]]` attribute lets you annotate function parameters as non-null
