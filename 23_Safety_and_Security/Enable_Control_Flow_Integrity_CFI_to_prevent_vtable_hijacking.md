# Enable Control Flow Integrity (CFI) to prevent vtable hijacking

**Category:** Safety & Security  
**Item:** #560  
**Reference:** <https://clang.llvm.org/docs/ControlFlowIntegrity.html>  

---

## Topic Overview

CFI ensures indirect calls (virtual calls, function pointers) only reach valid targets. This prevents vtable hijacking, ROP gadgets, and type confusion attacks. Complementary to #736 (more CFI details) and #656 (CFI + vtable hijacking attacks).

```cpp

Vtable hijacking attack:

  Normal:                         Attack:
  obj.vptr -> [vtable]            obj.vptr -> [fake_vtable]  (overwritten)
               |                                |
           [Derived::f()]                  [shellcode / gadget]

  CFI protection:
  Before indirect call:

    1. Check: does target address belong to a valid function?
    2. Check: is the function the RIGHT TYPE for this call site?
    3. If not: trap / abort

```

| CFI scheme | Flag | Protects |
| --- | --- | --- |
| vcall | `-fsanitize=cfi-vcall` | Virtual function calls |
| nvcall | `-fsanitize=cfi-nvcall` | Non-virtual member calls |
| derived-cast | `-fsanitize=cfi-derived-cast` | `static_cast` to derived |
| unrelated-cast | `-fsanitize=cfi-unrelated-cast` | Casts between unrelated types |
| icall | `-fsanitize=cfi-icall` | Indirect calls via function pointer |
| All | `-fsanitize=cfi` | All of the above |

---

## Self-Assessment

### Q1: Compile with -fsanitize=cfi

```cpp

// Compile: clang++ -std=c++20 -flto -fvisibility=hidden -fsanitize=cfi cfi_demo.cpp
// Note: CFI requires LTO (-flto) and hidden visibility (-fvisibility=hidden)
#include <iostream>

class Base {
public:
    virtual void greet() { std::cout << "Base::greet\n"; }
    virtual ~Base() = default;
};

class Derived : public Base {
public:
    void greet() override { std::cout << "Derived::greet\n"; }
};

class Unrelated {
public:
    virtual void hack() { std::cout << "Unrelated::hack\n"; }
};

int main() {
    // Normal virtual call - CFI allows this
    Base* b = new Derived();
    b->greet();  // OK: Derived IS-A Base

    // Type confusion attack - CFI catches this!
    // Attacker corrupts vptr to point to Unrelated's vtable
    Unrelated* u = new Unrelated();

    // Simulating vtable hijack (reinterpret_cast)
    // Base* fake = reinterpret_cast<Base*>(u);
    // fake->greet();  // CFI TRAP: target type doesn't match!

    // With CFI: program aborts before the call executes
    // Without CFI: calls Unrelated::hack() with wrong 'this' -> exploit

    std::cout << "\nCFI protects:\n";
    std::cout << "  1. Virtual calls (cfi-vcall): vtable pointer validation\n";
    std::cout << "  2. Indirect calls (cfi-icall): function pointer type check\n";
    std::cout << "  3. Bad casts (cfi-derived-cast): static_cast<Derived*>(base)\n";

    delete b;
    delete u;
}

```

### Q2: How CFI validates function pointers

CFI uses type metadata embedded at compile time (via LTO) to validate targets:

**Step 1: Compile with LTO** — The compiler sees ALL code, can build a type graph.

**Step 2: Type metadata** — Each function/vtable gets a type identifier based on its signature.

**Step 3: Before indirect call** — Compiler inserts a check:

```cpp

// Pseudocode of CFI check:
if (target_address NOT in valid_targets_for_type(expected_type))
    __cfi_check_fail();  // trap!
else
    call target_address;

```

**Implementation detail (Clang):**

- Uses bitset or range check per call site type.
- Valid targets stored in a compact data structure in the binary.
- Check is ~2-3 instructions: load type metadata, compare, branch.

**What CFI catches:**

- Corrupted vtable pointer → points to non-vtable memory.
- Type confusion → vtable for wrong class hierarchy.
- Corrupted function pointer → points to wrong function type.
- JIT-sprayed shellcode → not in valid function set.

**What CFI does NOT catch:**

- Calling a valid function of the right type but wrong instance.
- Data-only attacks (not involving indirect calls).
- Intra-function control flow (use CFG for that).

### Q3: CFI overhead and production suitability

| Metric | Without CFI | With CFI |
| --- | --- | --- |
| Binary size | Baseline | +5-15% (type metadata) |
| Runtime overhead | Baseline | +1-5% (typical) |
| Virtual call cost | ~1 indirect jump | ~3 instructions + jump |
| Compilation time | Baseline | +20-50% (requires LTO) |

**Production deployment:**

```cpp

# Build command for production CFI:
clang++ -std=c++20 -O2 \
  -flto                          \ # Required: Link-Time Optimization
  -fvisibility=hidden            \ # Required: default hidden visibility
  -fsanitize=cfi                 \ # Enable all CFI checks
  -fno-sanitize-recover=cfi      \ # Abort on violation (don't continue)
  main.cpp -o app

# For diagnostics (development):
clang++ ... -fsanitize=cfi -fsanitize-recover=cfi \
  -fno-sanitize-trap=cfi         \ # Print diagnostic instead of trap

```

**When to enable CFI in production:**

- Security-critical applications (browsers, OS components).
- Code handling untrusted input (parsers, network servers).
- Already: Chrome, Android, Linux kernel use CFI.

**When to skip:**

- Performance-critical inner loops (rare case where 1-5% matters).
- Heavy use of plugins/dlopen (CFI requires LTO across boundaries).
- MSVC (CFI is Clang-specific; MSVC has Control Flow Guard /guard:cf).

---

## Notes

- Complementary to #736 and #656 (more CFI topics).
- CFI REQUIRES `-flto` (Link-Time Optimization) and `-fvisibility=hidden`.
- MSVC equivalent: `/guard:cf` (Control Flow Guard) — different mechanism, similar goal.
- Android enables CFI by default for system binaries since Android 9.
- Chrome uses CFI for all builds since 2018.
