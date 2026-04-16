# Use time-travel debugging with rr record and replay

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://rr-project.org/>  

---

## Topic Overview

`rr` records a program's execution deterministically, then lets you replay it forwards and backwards. This is invaluable for debugging intermittent bugs, race conditions, and crashes.

### Basic Workflow

```bash

# Record the program (runs ~2x slower)
rr record ./my_program --arg1 --arg2

# Replay — the execution is byte-for-byte identical
rr replay

# Inside the replay GDB session:
(rr) continue            # Run forward
(rr) reverse-continue    # Run backward!
(rr) reverse-step        # Step backward one source line
(rr) reverse-next        # Step backward over function calls

# Example: find where a variable was corrupted
(rr) watch -l my_object.count   # Hardware watchpoint
(rr) reverse-continue           # Run backward to the PREVIOUS write
# → Stops at the exact instruction that corrupted the value

```

### Debugging Race Conditions

```bash

# rr records ALL threads deterministically
# Even data races reproduce the same way every replay

# Record a program that crashes 1 in 100 runs:
for i in $(seq 1 100); do
    rr record ./flaky_test && break
done
# Once recorded, that exact crash replays forever

# In replay:
(rr) info threads                    # See all threads
(rr) thread 3                        # Switch to thread 3
(rr) reverse-continue                # Go back in time on this thread
(rr) when                            # Show current event number
(rr) run 500                         # Jump to event 500

```

### Conditional Reverse Debugging

```bash

# Set a conditional watchpoint and go backward
(rr) watch my_map.size()
(rr) condition 1 my_map.size() > 1000
(rr) reverse-continue
# Stops at the moment size exceeded 1000 — going backwards in time

```

---

## Self-Assessment

### Q1: What are rr's limitations

- Linux only (uses `perf_event_open`)
- x86/x86-64 only (ARM support experimental)
- Single-machine only (no distributed recording)
- ~2x slowdown during recording
- Doesn't work well inside VMs without hardware perf counter passthrough
- Cannot record GPU operations or kernel-mode code

### Q2: How does rr achieve deterministic replay

rr records all sources of non-determinism: system call results, signal delivery, thread scheduling decisions, rdtsc values, and hardware interrupts. During replay, it intercepts these and feeds back the recorded values, ensuring byte-for-byte identical execution.

### Q3: Compare rr with other time-travel debuggers

| Debugger | Platform | Approach |
| --- | --- | --- |
| rr | Linux | Record/replay via perf counters |
| UDB (UndoDB) | Linux | Commercial, supports ARM |
| WinDbg TTD | Windows | Time Travel Debugging, Microsoft |
| QIRA | Linux | Full system emulation |

---

## Notes

- `rr` is free and open source — used by Mozilla, Chromium, and many other projects.
- Record once, debug many times: share the recording with teammates.
- `rr chaos` mode randomizes scheduling to find concurrency bugs faster.
- Combine with AddressSanitizer: `rr record ./asan_binary` to get time-travel.
