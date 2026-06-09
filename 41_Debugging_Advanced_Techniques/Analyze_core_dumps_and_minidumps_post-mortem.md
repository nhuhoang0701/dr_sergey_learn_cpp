# Analyze core dumps and minidumps post-mortem

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Core-File-Generation.html>  

---

## Topic Overview

When a program crashes in production, you usually don't have a debugger attached and you can't reproduce the crash on demand. That's exactly what core dumps and minidumps are for. They capture a snapshot of the entire process memory - every stack frame, every local variable, every object - at the exact moment of the crash. You can then load that snapshot later, on a different machine, and inspect the state as if you had been there when it happened.

This is called post-mortem debugging, and it's one of the most powerful tools in your production debugging toolkit. The key insight is that a core dump is a frozen moment in time: you can walk the stack, inspect variables, and examine memory as if the program were still running - even days after the crash occurred.

### Enabling Core Dumps

Before you can use core dumps, you need to tell the operating system to actually write them. By default on many Linux systems, core files are either disabled or written to inconvenient locations.

Here's how to enable them and verify that they work:

```bash
# Linux: enable core dumps
ulimit -c unlimited              # Allow unlimited core file size
echo "/tmp/core.%e.%p" > /proc/sys/kernel/core_pattern

# Verify: trigger a crash
kill -SIGSEGV $$

# systemd systems - cores go to coredumpctl:
coredumpctl list                 # List recent cores
coredumpctl gdb my_program       # Open most recent core for my_program
```

The `%e` and `%p` in the core pattern are format specifiers that get replaced with the executable name and process ID, so you end up with a clearly named file. On systemd-based systems, `coredumpctl` handles this automatically and even compresses the dumps.

### GDB Post-Mortem Analysis

Once you have a core file and the original binary (compiled with debug symbols), GDB can load both and let you explore the crash state interactively. You're not running the program - you're examining its frozen memory image.

Here's a typical post-mortem session:

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

`bt full` is the first command to reach for - it gives you the full call chain plus the local variables at each frame. If a thread other than thread 1 crashed, `info threads` shows you all of them so you can switch to the interesting one. The `x/16gx $rsp` command is a raw memory dump from the stack pointer, useful when everything else is unavailable.

### Windows Minidumps

On Windows, the equivalent of a core dump is a minidump. You produce it by installing a crash handler that calls `MiniDumpWriteDump` from the Windows debugging API. The code below sets up an unhandled exception filter that writes a `.dmp` file whenever the process crashes:

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

The `MiniDumpWithFullMemory` flag tells the API to include all readable memory pages - this makes the dump file larger but gives you complete visibility into the heap, stacks, and all objects. You can then open the `.dmp` file in WinDbg or Visual Studio and step through the crash just like a live debugging session.

---

## Self-Assessment

### Q1: What build settings produce useful core dumps

To get useful information from a core dump, you need to ship your binary with debug symbols - or at least keep those symbols accessible on a symbol server. Stripping the binary is fine for production (it reduces load time), but you need to save the debug information somewhere you can retrieve it later.

```cmake
# Ship with debug symbols in a separate file
target_compile_options(myapp PRIVATE -g -O2 -fno-omit-frame-pointer)

# Split debug info (keep binary small, symbols separate):
# objcopy --only-keep-debug myapp myapp.debug
# objcopy --strip-debug myapp
# objcopy --add-gnu-debuglink=myapp.debug myapp
# Ship myapp to customers, keep myapp.debug on your symbol server
```

The `-fno-omit-frame-pointer` flag is the important one here. Without it, the compiler is free to repurpose the frame pointer register for general computation, which breaks most stack-walking tools. Keep it on, even in optimized builds.

### Q2: How to analyze a core dump when the binary has no symbols

Even without symbols you're not completely blind. Load separate debug symbols with `(gdb) symbol-file /path/to/myapp.debug`. If no separate file exists at all, you can still examine the raw stack with `x/32gx $rsp`, look for recognizable values or addresses, and use `info sharedlib` to identify which shared libraries were loaded and at what addresses.

### Q3: What is a signal that caused the core dump

The signal number tells you the category of crash immediately, before you even look at the stack. Here's how to check:

```bash
(gdb) info signal
# Common crash signals:
# SIGSEGV (11) - null/bad pointer dereference
# SIGABRT (6)  - abort() called (assertion failure, uncaught exception)
# SIGFPE (8)   - floating-point exception (divide by zero)
# SIGBUS (7)   - misaligned memory access
```

SIGABRT is particularly useful because it typically means `abort()` or `std::terminate()` was called, which happens on assertion failures and uncaught exceptions - both of which usually leave a clear reason in the stack trace.

---

## Notes

- Always compile with `-fno-omit-frame-pointer` for production binaries - it enables reliable stack traces.
- Use `coredumpctl` on systemd Linux - it compresses and retains cores automatically.
- Symbol servers (debuginfod, Microsoft Symbol Server) provide symbols on demand.
- On Windows, WinDbg with `.ecxr` shows the exception context from a minidump.
