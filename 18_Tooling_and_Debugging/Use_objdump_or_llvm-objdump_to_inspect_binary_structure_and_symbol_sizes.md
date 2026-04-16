# Use objdump or llvm-objdump to inspect binary structure and symbol sizes

**Category:** Tooling & Debugging  
**Item:** #420  
**Reference:** <https://llvm.org/docs/CommandGuide/llvm-objdump.html>  

---

## Topic Overview

Inspecting compiled binaries helps understand what the compiler produced, debug linking issues, and track binary size.

```cpp

ELF binary structure:
┌────────────────────────┐
│  ELF Header            │
├────────────────────────┤
│  .text (code)          │  ← executable instructions
├────────────────────────┤
│  .rodata (read-only)   │  ← string literals, const data
├────────────────────────┤
│  .data (initialized)   │  ← global/static initialized vars
├────────────────────────┤
│  .bss (uninitialized)  │  ← zero-initialized globals (no file space)
├────────────────────────┤
│  .symtab (symbols)     │  ← function/variable names + sizes
└────────────────────────┘

```

---

## Self-Assessment

### Q1: Find which symbols contribute most to binary size

```bash

# nm: list all symbols with sizes, sorted largest first
$ nm --size-sort --reverse-sort --demangle myapp | head -15
# 00000000000045a0 0000089c T void std::__introsort_loop<...>(...)
# 00000000000023b0 00000567 T process_data(std::vector<int>&)
# 0000000000001230 00000345 T main
#

# Filter only defined text (code) symbols:
$ nm --size-sort --demangle myapp | grep ' [Tt] ' | tail -10

# Total code size from all symbols:
$ nm --size-sort -t d myapp | awk '/[Tt]/{sum+=$1} END{print sum " bytes of code"}'

# objdump: section headers with sizes
$ objdump -h myapp
# Idx Name          Size      VMA
#   1 .text         0000abc0  0000000000401000
#   2 .rodata       00001234  000000000040bc00
#   3 .data         00000100  000000000060d000
#   4 .bss          00000200  000000000060d100

```

### Q2: Measure text/data/bss with `llvm-size`

```bash

# Basic section sizes:
$ llvm-size myapp
#    text    data     bss     dec     hex filename
#   43968    1024     512   45504    b1c0 myapp

# Berkeley format (default) vs System V format:
$ llvm-size --format=sysv myapp
# myapp  :
# section          size     addr
# .text           43968   4198400
# .rodata          4660   4242432
# .data            1024   6316032
# .bss              512   6317056
#

# Compare multiple objects:
$ llvm-size *.o
#    text    data     bss     dec     hex filename
#    2345     100       0    2445     98d main.o
#    5678     200      16    5894    1706 util.o
#    8901     300      32    9233    2411 math.o

# GNU size equivalent:
$ size myapp

```

### Q3: Detect template instantiation explosion

```cpp

// bloat.cpp — generates many template instantiations
#include <vector>
#include <list>
#include <deque>
#include <map>
#include <set>
#include <string>

// Each unique type creates a new instantiation:
template<typename T>
void process(T& container) {
    for (auto& x : container)
        x = {};
}

int main() {
    std::vector<int> v1(10);
    std::vector<double> v2(10);
    std::vector<std::string> v3(10);
    std::list<int> l1;
    std::list<double> l2;
    std::deque<int> d1;
    std::map<int, std::string> m1;
    std::set<double> s1;

    process(v1); process(v2); process(v3);
    process(l1); process(l2);
    process(d1); process(m1); process(s1);
}

```

```bash

# Find template bloat by sorting symbols by size:
$ nm --size-sort --demangle bloat | grep 'process<' | sort -rn
# 00000345 T void process<std::map<int, std::string>>(std::map<...>&)
# 00000234 T void process<std::vector<std::string>>(std::vector<...>&)
# 00000189 T void process<std::list<double>>(std::list<...>&)
# 00000145 T void process<std::vector<int>>(std::vector<...>&)
# ...8 instantiations

# Count unique template instantiations:
$ nm --demangle bloat | grep 'process<' | wc -l
# 8

# Total size of template instantiations:
$ nm --size-sort -t d --demangle bloat | grep 'process<' | awk '{sum+=$1} END{print sum}'
# 2345 bytes just for process<> variants

# Fix: use type erasure or explicit instantiation to reduce bloat

```

---

## Notes

- `c++filt` demangles symbol names: `echo _Z7processi | c++filt` → `process(int)`.
- `.bss` doesn't take file space but occupies memory at runtime.
- `strip --strip-all` removes symbols, reducing file size (~30-50%).
- On macOS, use `otool -l` for section info and `otool -tV` for disassembly.
- Template instantiation bloat is the #1 cause of large C++ binaries.
