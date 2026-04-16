# Debug across library boundaries with mixed debug/release and symbol servers

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/elfutils/Debuginfod.html>  

---

## Topic Overview

Real projects link many libraries, often compiled with different compilers or optimization levels. Debugging across these boundaries requires symbol servers and careful configuration.

### Mixed Debug/Release Issues

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

### Linux: debuginfod

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

### Windows: Symbol Servers

```cpp

# In Visual Studio or WinDbg:
# Tools → Options → Debugging → Symbols
# Add symbol server: https://msdl.microsoft.com/download/symbols

# WinDbg: .sympath+ srv*c:\symbols*https://mycompany.com/symbols

# Upload PDB files to your symbol server:
# symstore add /r /f *.pdb /s \\server\symbols /t "MyApp" /v "2.1"

```

### CMake: Split Debug Info

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

---

## Self-Assessment

### Q1: What happens when you step into a function compiled without debug info

The debugger shows raw assembly. Local variables aren't available. The call stack frame shows the function address and symbol name (if visible) but no source file or line number. With separate debug symbols, this is fixed transparently.

### Q2: How does GNU debug link work

`--add-gnu-debuglink` embeds a filename and CRC32 of the debug file into the stripped binary. When GDB loads the stripped binary, it searches for the debug file by name in standard locations (`/usr/lib/debug/`, `<binary-dir>/.debug/`, etc.) and verifies the CRC.

### Q3: How to debug when libraries have mismatched C++ standard library versions

Different libraries linked with different libstdc++ versions may have incompatible `std::string` (pre/post CXX11 ABI). Use `nm -C library.so | grep "__cxx11"` to check ABI version. Ensure all libraries use the same ABI or use `extern "C"` boundaries.

---

## Notes

- Always store debug symbols (.debug/.pdb) for every release build — you WILL need them.
- `debuginfod` is the Linux equivalent of Microsoft Symbol Server — set it up for your org.
- `-gsplit-dwarf` (GCC/Clang) produces `.dwo` files for faster linking.
- Mixed debug/release builds mostly work but beware of ABI differences in debug iterators.
