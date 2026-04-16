# Remote debug embedded targets with GDB server

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Remote-Debugging.html>  

---

## Topic Overview

Embedded targets (microcontrollers, remote Linux boards) can't run a full debugger. GDB server runs on/near the target, and GDB connects from your development machine.

### Architecture

```cpp

Development Machine              Target Device
┌─────────────────┐             ┌──────────────┐
│  GDB (client)   │──TCP/USB──▶│  gdbserver    │
│  with symbols   │             │  (stub)       │
│  and source     │             │  + your app   │
└─────────────────┘             └──────────────┘

For bare-metal: Use OpenOCD / J-Link GDB Server instead of gdbserver

```

### Linux Remote Target

```bash

# On target (embedded Linux):
gdbserver :2345 ./my_program arg1 arg2
# Or attach to running process:
gdbserver :2345 --attach 1234

# On development machine:
arm-linux-gnueabihf-gdb ./my_program    # Cross-GDB with symbols
(gdb) target remote 192.168.1.100:2345
(gdb) break main
(gdb) continue
(gdb) bt
# Full debugging: breakpoints, watchpoints, step, print variables

```

### Bare-Metal with OpenOCD

```bash

# Start OpenOCD (connects to J-Link/ST-Link/CMSIS-DAP):
openocd -f interface/stlink.cfg -f target/stm32f4x.cfg

# Connect GDB:
arm-none-eabi-gdb firmware.elf
(gdb) target remote :3333
(gdb) monitor reset halt       # Reset and halt the MCU
(gdb) load                     # Flash firmware.elf to the target
(gdb) break main
(gdb) continue

# Hardware breakpoints (limited on MCUs):
(gdb) hbreak my_function       # Uses hardware breakpoint register
(gdb) info break               # Shows hw vs sw breakpoints

```

### VS Code Integration

```json

// .vscode/launch.json
{
    "name": "Remote Debug",
    "type": "cppdbg",
    "request": "launch",
    "program": "${workspaceFolder}/build/firmware.elf",
    "MIMode": "gdb",
    "miDebuggerPath": "arm-none-eabi-gdb",
    "miDebuggerServerAddress": "localhost:3333",
    "setupCommands": [
        {"text": "target remote :3333"},
        {"text": "monitor reset halt"},
        {"text": "load"}
    ]
}

```

---

## Self-Assessment

### Q1: What's the difference between `target remote` and `target extended-remote`

`target remote`: GDB connects to one session. When the program exits, the connection closes. `target extended-remote`: GDB can restart the program, run multiple processes, and disconnect/reconnect without killing the debug session.

### Q2: How many hardware breakpoints do typical MCUs support

ARM Cortex-M3/M4: 6 hardware breakpoints, 4 hardware watchpoints. ARM Cortex-M0: 2 breakpoints, 2 watchpoints. Software breakpoints (replacing instruction with BKPT) need writable memory and are unlimited.

### Q3: How to debug with semihosting

```bash

# Semihosting routes stdout/stdin from MCU through the debugger
(gdb) monitor arm semihosting enable
# Now printf() in firmware outputs to your GDB console
# Useful for embedded logging without UART

```

---

## Notes

- Cross-compile with `-g` and debug symbols for the target architecture.
- `gdbserver` on Linux is lightweight (~100KB) — suitable even for small embedded Linux devices.
- For real-time systems, use non-stop mode: `set non-stop on` (doesn't halt all threads).
- J-Link's GDB server supports SWO trace output alongside debugging.
