# Debug across library boundaries with mixed debug/release and symbol servers

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/elfutils/Debuginfod.html>  

---

## Topic Overview

In any real project, your application links against libraries you didn't build yourself - system libraries, third-party SDKs, or internal shared components compiled by a different team. These libraries are almost always compiled with release settings, while your own code is compiled with debug symbols. Debugging across that boundary is genuinely awkward, and understanding the tools that exist for it saves a lot of pain.

The core problem is: when you step into a library call in your debugger, you land in raw assembly with no source lines and no variable names. This is the mixed debug/release scenario.

### Mixed Debug/Release Issues

Let's be concrete about what "mixed" means and what the fixes are:

```cpp
Problem: Your debug app links against a release library.

- Library functions show as assembly (no source lines)
- Stepping into library calls jumps to disassembly
- Variables in library frames show as "optimized out"

Solutions:

1. Separate debug symbols (.debug / .pdb files)
2. Symbol servers (debuginfod / Microsoft Symbol Server)
3. Debug builds of all dependencies (slow but complete)
```

Option 3 - rebuilding all dependencies in debug mode - is the nuclear option. It gives you full visibility but takes much longer to build. Options 1 and 2 are what you'll use in practice: you keep the release binary as-is, but make the debug symbols available through a symbol server so the debugger can fetch them transparently.

### Linux: debuginfod

`debuginfod` is a simple HTTP server that serves debug symbols, source files, and executables on demand. GDB queries it automatically when it can't find symbols locally. For system libraries on common Linux distributions, the public `debuginfod.elfutils.org` service already has most of what you need.

Here's how to point GDB at it:

```bash
# debuginfod automatically downloads debug symbols for system libraries
export DEBUGINFOD_URLS="https://debuginfod.elfutils.org/"

# GDB automatically queries debuginfod for missing symbols
gdb ./my_program
(gdb) bt
# Previously: libc.so.6`__libc_start_main+0xf0 (no source info)
# Now: ../csu/libc-start.c:314 (source available!)

# Self-hosted debuginfod for your organization:
debuginfod -F /opt/debug-symbols/ -p 8002
# Serves .debug files, source files, and executables
```

The transformation in the `bt` output above - from a raw address to an actual source file and line number - is what symbol servers buy you. For your own internal libraries, you host your own `debuginfod` instance and point it at your collection of `.debug` files.

### Windows: Symbol Servers

Windows has had the same concept since the early 2000s through Microsoft's symbol server infrastructure. Both Visual Studio and WinDbg support it natively.

```cpp
# In Visual Studio or WinDbg:
# Tools -> Options -> Debugging -> Symbols
# Add symbol server: https://msdl.microsoft.com/download/symbols

# WinDbg: .sympath+ srv*c:\symbols*https://mycompany.com/symbols

# Upload PDB files to your symbol server:
# symstore add /r /f *.pdb /s \\server\symbols /t "MyApp" /v "2.1"
```

The `msdl.microsoft.com` server contains PDB files for all released Windows system DLLs, so you can step through `ntdll.dll` or `kernel32.dll` code with full source. For your own company, `symstore` is the standard tool for uploading PDB files to an internal share.

### CMake: Split Debug Info

The standard workflow on Linux is to build with full debug info, then split the symbols out into a separate `.debug` file before shipping the binary. The CMake function below does this as a post-build step:

```cmake
function(split_debug_info target)
    if(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        add_custom_command(TARGET ${target} POST_BUILD
            COMMAND ${CMAKE_OBJCOPY} --only-keep-debug
                    $<TARGET_FILE:${target}>
                    $<TARGET_FILE:${target}>.debug
            COMMAND ${CMAKE_OBJCOPY} --strip-debug $<TARGET_FILE:${target}>
            COMMAND ${CMAKE_OBJCOPY} --add-gnu-debuglink=$<TARGET_FILE:${target}>.debug
                    $<TARGET_FILE:${target}>
        )
    endif()
endfunction()

split_debug_info(myapp)
# Result: myapp (small, stripped), myapp.debug (symbols)
```

The `--add-gnu-debuglink` step is what links the two files together. GDB reads the stripped binary, finds the debug link pointing to `myapp.debug`, and loads the symbols automatically. Ship `myapp` to customers; archive `myapp.debug` on your symbol server.

---

## Self-Assessment

### Q1: What happens when you step into a function compiled without debug info

The debugger drops you into raw assembly. Local variables aren't available - they may exist in registers or memory, but without DWARF information the debugger doesn't know how to interpret them. The call stack frame shows the function address and symbol name if the binary wasn't fully stripped, but no source file or line number. With separate debug symbols on a symbol server, this is fixed transparently - GDB fetches the symbols and suddenly you have source lines again.

### Q2: How does GNU debug link work

`--add-gnu-debuglink` embeds a filename and CRC32 of the debug file into the stripped binary. When GDB loads the stripped binary, it reads this embedded reference, then searches for the debug file by name in standard locations (`/usr/lib/debug/`, `<binary-dir>/.debug/`, etc.) and verifies the file against the embedded CRC. If the CRC doesn't match - meaning the debug file is from a different build than the binary - GDB rejects it, which protects you from accidentally loading mismatched symbols.

### Q3: How to debug when libraries have mismatched C++ standard library versions

Different libraries linked with different `libstdc++` versions may have incompatible `std::string` layouts - this is the CXX11 ABI break (pre/post GCC 5). Use `nm -C library.so | grep "__cxx11"` to check which ABI version a library was built against. If you're seeing strange crashes or corrupted string data when crossing library boundaries, ABI mismatch is a strong suspect. The only clean fix is to ensure all libraries use the same ABI, or to use `extern "C"` boundaries between components that may have been compiled with different settings.

---

## Notes

- Always store debug symbols (.debug/.pdb) for every release build - you WILL need them.
- `debuginfod` is the Linux equivalent of Microsoft Symbol Server - set it up for your org.
- `-gsplit-dwarf` (GCC/Clang) produces `.dwo` files for faster linking.
- Mixed debug/release builds mostly work but beware of ABI differences in debug iterators.
