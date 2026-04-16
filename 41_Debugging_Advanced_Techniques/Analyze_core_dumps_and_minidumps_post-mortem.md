# Analyze core dumps and minidumps post-mortem

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Core-File-Generation.html>  

---

## Topic Overview

Core dumps capture the process memory state at crash time. Post-mortem analysis lets you debug crashes that happen in production without a debugger attached.

### Enabling Core Dumps

```bash

# Linux: enable core dumps
ulimit -c unlimited              # Allow unlimited core file size
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern

# Verify: trigger a crash
kill -SIGSEGV $$

# systemd systems — cores go to coredumpctl:
coredumpctl list                 # List recent cores
coredumpctl gdb my_program       # Open most recent core for my_program

```

### GDB Post-Mortem Analysis

```bash

# Load core dump with symbols
gdb ./my_program /tmp/core.my_program.12345

(gdb) bt               # Backtrace of crashing thread
(gdb) bt full           # Backtrace with local variables
(gdb) info threads      # List all threads at crash time
(gdb) thread 3          # Switch to thread 3
(gdb) bt                # See thread 3's backtrace
(gdb) frame 2           # Go to frame 2
(gdb) info locals       # Print local variables in that frame
(gdb) print *this       # Print the object
(gdb) x/16gx $rsp      # Examine 16 8-byte values at stack pointer

```

### Windows Minidumps

```cpp

#include <windows.h>
#include <dbghelp.h>
#pragma comment(lib, "dbghelp.lib")

LONG WINAPI CrashHandler(EXCEPTION_POINTERS* pExceptionInfo) {
    HANDLE hFile = CreateFileW(L"crash.dmp", GENERIC_WRITE, 0,
                               nullptr, CREATE_ALWAYS, 0, nullptr);
    MINIDUMP_EXCEPTION_INFORMATION mei;
    mei.ThreadId = GetCurrentThreadId();
    mei.ExceptionPointers = pExceptionInfo;
    mei.ClientPointers = FALSE;
    MiniDumpWriteDump(GetCurrentProcess(), GetCurrentProcessId(),
                      hFile, MiniDumpWithFullMemory, &mei, nullptr, nullptr);
    CloseHandle(hFile);
    return EXCEPTION_EXECUTE_HANDLER;
}

int main() {
    SetUnhandledExceptionFilter(CrashHandler);
    // ... program code
}

```

---

## Self-Assessment

### Q1: What build settings produce useful core dumps

```cmake

# Ship with debug symbols in a separate file
target_compile_options(myapp PRIVATE -g -O2 -fno-omit-frame-pointer)

# Split debug info (keep binary small, symbols separate):
# objcopy --only-keep-debug myapp myapp.debug
# objcopy --strip-debug myapp
# objcopy --add-gnu-debuglink=myapp.debug myapp
# Ship myapp to customers, keep myapp.debug on your symbol server

```

### Q2: How to analyze a core dump when the binary has no symbols

Load separate debug symbols: `(gdb) symbol-file /path/to/myapp.debug`. If no separate file exists, you can still examine the raw stack (`x/32gx $rsp`), look for known values, and use `info sharedlib` to identify loaded libraries and their offsets.

### Q3: What is a signal that caused the core dump

```bash

(gdb) info signal
# Common crash signals:
# SIGSEGV (11) — null/bad pointer dereference
# SIGABRT (6)  — abort() called (assertion failure, uncaught exception)
# SIGFPE (8)   — floating-point exception (divide by zero)
# SIGBUS (7)   — misaligned memory access

```

---

## Notes

- Always compile with `-fno-omit-frame-pointer` for production binaries — it enables reliable stack traces.
- Use `coredumpctl` on systemd Linux — it compresses and retains cores automatically.
- Symbol servers (debuginfod, Microsoft Symbol Server) provide symbols on demand.
- On Windows, WinDbg with `.ecxr` shows the exception context from a minidump.
