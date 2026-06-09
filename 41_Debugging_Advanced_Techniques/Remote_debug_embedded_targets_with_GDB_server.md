# Remote debug embedded targets with GDB server

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://sourceware.org/gdb/current/onlinedocs/gdb.html/Remote-Debugging.html>  

---

## Topic Overview

Embedded targets - microcontrollers, remote Linux boards, devices with no keyboard or monitor - can't run a full debugger. They don't have the memory, the OS infrastructure, or sometimes even a general-purpose OS at all. The solution is a split architecture: a lightweight stub called `gdbserver` runs on or near the target and handles the low-level operations (reading memory, setting breakpoints, reporting execution state), while GDB runs on your development machine where you have full resources. The two talk over a TCP connection or a USB cable. You get the same debugging experience you'd have locally, but the program under test is running on the embedded hardware.

### Architecture

The split looks like this. Your development machine runs a cross-compiled GDB that has all the debug symbols and source files. The target device runs only the minimal `gdbserver` stub:

```cpp
Development Machine              Target Device
┌─────────────────┐             ┌──────────────┐
│  GDB (client)   │--TCP/USB-->│  gdbserver    │
│  with symbols   │             │  (stub)       │
│  and source     │             │  + your app   │
└─────────────────┘             └──────────────┘

For bare-metal: Use OpenOCD / J-Link GDB Server instead of gdbserver
```

For bare-metal targets (microcontrollers with no OS), there's no process model to attach to, so `gdbserver` isn't used. Instead, OpenOCD or a J-Link GDB Server acts as the bridge, communicating with the chip over SWD or JTAG.

### Linux Remote Target

If your target is running embedded Linux (a Raspberry Pi, a BeagleBone, a custom board with a full Linux distribution), `gdbserver` is available as a normal package. You start it on the target and point your cross-compiled GDB at it from your machine:

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

The key is using the cross-compiled GDB (note the `arm-linux-gnueabihf-gdb` prefix) because it knows how to interpret ARM binaries and debug information. The binary you pass to GDB on your development machine is the same ELF file you deployed to the target - it contains all the symbols and source paths.

### Bare-Metal with OpenOCD

For microcontrollers, OpenOCD is the most common open-source solution. It translates between GDB's remote protocol and the SWD/JTAG signals that actually talk to the chip. You launch OpenOCD pointing at your debug adapter (ST-Link, J-Link, CMSIS-DAP), then connect GDB to the OpenOCD port:

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

The `monitor reset halt` and `load` commands are important for embedded work. `monitor reset halt` tells OpenOCD to reset the chip and immediately halt it before it starts running, giving you a clean state. `load` flashes your firmware to the target's flash memory. These steps replace the usual "the OS loads your binary" that you take for granted on a desktop.

Notice `hbreak` for hardware breakpoints. On microcontrollers, hardware breakpoints are a limited resource - the CPU dedicates a small number of debug registers for them. Software breakpoints work by replacing an instruction with a breakpoint instruction in memory, which only works if the memory is writable (flash usually isn't at runtime). Hardware breakpoints don't modify memory at all, so they work anywhere.

### VS Code Integration

For day-to-day embedded work, you don't want to type GDB commands manually every session. A `launch.json` in VS Code wires up the same connection automatically, so pressing F5 resets the target, flashes the firmware, and drops you at `main`:

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

The `setupCommands` list runs the same bare-metal initialization sequence you'd type manually. Once this is configured, your team members get the same setup without needing to know the underlying GDB commands.

---

## Self-Assessment

### Q1: What's the difference between `target remote` and `target extended-remote`

`target remote` connects GDB to a single session. When the program exits or the connection drops, the GDB session ends and you have to start over. `target extended-remote` is more capable: GDB can restart the program on the remote side, manage multiple processes, and disconnect and reconnect without killing the debug session. For embedded Linux development where you're frequently relaunching a program, `extended-remote` is usually more convenient. For bare-metal with OpenOCD, `target remote` is more common because the concept of "restarting a program" maps to reflashing the MCU.

### Q2: How many hardware breakpoints do typical MCUs support

This varies by core type, and running out of hardware breakpoints is a real constraint you'll hit when debugging embedded code:

ARM Cortex-M3/M4: 6 hardware breakpoints, 4 hardware watchpoints. ARM Cortex-M0: 2 breakpoints, 2 watchpoints. Software breakpoints (replacing an instruction with `BKPT`) need writable memory and are essentially unlimited, but they don't work in flash-resident code unless you've set up a flash patch unit. When you need more breakpoints than the hardware provides, GDB silently falls back to software breakpoints where it can.

### Q3: How to debug with semihosting

Semihosting is a technique that lets code running on the MCU use I/O facilities from the host debugger. The MCU executes a special breakpoint instruction that OpenOCD or the GDB server intercepts, routes to the host, and handles. The most useful application is making `printf` work without a UART:

```bash
# Semihosting routes stdout/stdin from MCU through the debugger
(gdb) monitor arm semihosting enable
# Now printf() in firmware outputs to your GDB console
# Useful for embedded logging without UART
```

This is particularly handy early in a project when you don't yet have a UART driver working but you want some way to see output from the firmware.

---

## Notes

- Cross-compile with `-g` and debug symbols for the target architecture.
- `gdbserver` on Linux is lightweight (~100KB) - suitable even for small embedded Linux devices.
- For real-time systems, use non-stop mode: `set non-stop on` (doesn't halt all threads).
- J-Link's GDB server supports SWO trace output alongside debugging.
