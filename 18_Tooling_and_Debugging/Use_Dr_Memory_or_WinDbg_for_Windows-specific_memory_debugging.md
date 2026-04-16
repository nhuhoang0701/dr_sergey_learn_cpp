# Use Dr. Memory or WinDbg for Windows-specific memory debugging

**Category:** Tooling & Debugging  
**Item:** #295  
**Reference:** <https://drmemory.org>  

---

## Topic Overview

Windows has specialized memory debugging tools complementary to Valgrind (Linux):

| Tool | Purpose | Approach |
| --- | --- | --- |
| **Dr. Memory** | Detect memory errors (leaks, overflows, use-after-free) | Dynamic binary instrumentation |
| **WinDbg** | Crash dump analysis, kernel debugging | Microsoft debugger |
| **PageHeap (gflags)** | Detect heap corruption at point of failure | Guard pages around allocations |
| **Application Verifier** | Broad error class detection | Runtime checks |

---

## Self-Assessment

### Q1: Run Dr. Memory and interpret a heap overwrite report

```cpp

// heap_overwrite.cpp — compile with debug info:
// cl /Zi /Od heap_overwrite.cpp  (MSVC)
// g++ -g -O0 heap_overwrite.cpp -o heap_overwrite.exe  (MinGW)
#include <cstdlib>
#include <iostream>

int main() {
    int* arr = new int[5];
    for (int i = 0; i < 5; ++i)
        arr[i] = i * 10;

    arr[5] = 999;   // BUG: heap buffer overwrite (1 past end)
    arr[-1] = -1;   // BUG: underwrite

    std::cout << arr[0] << '\n';
    delete[] arr;
}

```

```powershell

# Run with Dr. Memory:
drmemory.exe -- heap_overwrite.exe

# Dr. Memory output:
# ~~Dr.M~~ Error #1: UNADDRESSABLE ACCESS beyond heap bounds
# ~~Dr.M~~ writing 4 byte(s) at 0x01234568
# ~~Dr.M~~ refers to 0 byte(s) beyond a 20-byte malloc
# ~~Dr.M~~    # 0 main          [heap_overwrite.cpp:10]
#
# ~~Dr.M~~ Error #2: UNADDRESSABLE ACCESS before heap bounds
# ~~Dr.M~~ writing 4 byte(s) at 0x01234550
# ~~Dr.M~~    # 0 main          [heap_overwrite.cpp:11]

# Reading the report:
# - "20-byte malloc" = 5 ints * 4 bytes = 20 bytes (correct)
# - "0 byte(s) beyond" = writing right at the end boundary
# - File:line tells you exactly where the bug is

# Common Dr. Memory options:
drmemory.exe -show_reachable -- myapp.exe    # show reachable leaks
drmemory.exe -light -- myapp.exe             # faster, fewer checks
drmemory.exe -logdir ./logs -- myapp.exe     # custom log directory

```

### Q2: Use WinDbg `!analyze -v` to diagnose a crash dump

```cpp

// crash_example.cpp
#include <iostream>

void cause_crash() {
    int* p = nullptr;
    *p = 42;  // Access violation
}

int main() {
    std::cout << "About to crash...\n";
    cause_crash();
}

```

```powershell

# Step 1: Enable crash dumps via Windows Error Reporting
# Registry (or use procdump):
# HKLM\SOFTWARE\Microsoft\Windows\Windows Error Reporting\LocalDumps
# DumpType = 2 (Full dump)
#
# Or use procdump (Sysinternals):
# procdump -ma -e crash_example.exe

# Step 2: Open dump in WinDbg
# File -> Open Crash Dump -> crash_example.exe.dmp

# Step 3: Automatic analysis:
0:000> !analyze -v
# EXCEPTION_RECORD:  ACCESS_VIOLATION writing address 0x00000000
# FAULTING_IP:
#   crash_example!cause_crash+0x12
#     mov dword ptr [rax], 2Ah     ; 2Ah = 42 decimal
#
# STACK_TEXT:
#   crash_example!cause_crash+0x12  [crash_example.cpp @ 5]
#   crash_example!main+0x28         [crash_example.cpp @ 10]
#   kernel32!BaseThreadInitThunk
#
# SYMBOL_NAME: crash_example!cause_crash+12

# Step 4: Inspect variables
0:000> dv              ; display local variables
#   p = 0x00000000     ; confirmed null

0:000> k               ; stack trace
0:000> !heap -s        ; heap summary
0:000> !address -summary  ; memory summary

```

### Q3: Set up PageHeap to detect heap corruption at point of failure

```powershell

# PageHeap places guard pages around heap allocations
# so overflows cause immediate access violations (not silent corruption)

# Enable full PageHeap for an application:
gflags /p /enable myapp.exe /full

# Verify it's enabled:
gflags /p
# Output: myapp.exe: page heap enabled

# Now run the application — any heap overwrite triggers
# an immediate access violation at the exact instruction

# Disable when done (PageHeap adds significant overhead):
gflags /p /disable myapp.exe

```

```cpp

// This corruption would be caught IMMEDIATELY with PageHeap:
#include <cstdlib>
#include <cstring>

int main() {
    char* buf = new char[16];
    std::memset(buf, 'A', 32);  // Write 32 bytes into 16-byte buffer
                                 // Without PageHeap: silent corruption, crash later
                                 // With PageHeap: IMMEDIATE access violation HERE
    delete[] buf;
}

```

```cpp

PageHeap memory layout:
┌───────────┬────────────────┬─────────────┬────────────┐
│ Guard     │ Your allocation │ Padding       │ Guard page │
│ page (NA) │ (16 bytes)      │ (alignment)   │ (NA)       │
└───────────┴────────────────┴─────────────┴────────────┘
  ^                                               ^
  Write before buffer =                          Write past buffer =
  immediate AV                                    immediate AV

```

---

## Notes

- Dr. Memory works with both MSVC and MinGW binaries.
- PageHeap uses ~2x memory and is slower; enable only for debugging.
- WinDbg Preview (from Microsoft Store) has a modern UI.
- `!heap -stat -h 0` in WinDbg shows per-heap allocation statistics.
- Application Verifier (`appverif.exe`) adds checks for handles, locks, and TLS.
