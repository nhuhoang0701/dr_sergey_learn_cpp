# Use time-travel debugging with rr record and replay

**Category:** Debugging Advanced Techniques  
**Standard:** C++17  
**Reference:** <https://rr-project.org/>  

---

## Topic Overview

There is a class of bug that resists ordinary debugging completely: the kind that crashes only once in a hundred runs, or the kind where a variable is already corrupted by the time you attach a debugger and set a watchpoint. The ordinary approach is to add logging, rerun, and hope the bug reproduces in the same place. Often it doesn't.

`rr` solves this by separating the two concerns: first you *record* the execution (once), and then you *replay* it as many times as you want. The replay is byte-for-byte identical to the recording. Every system call, every thread schedule, every hardware interrupt happens in exactly the same order. This means once you capture a crash, you can debug it forever - and you can step *backwards* through execution, which changes debugging fundamentally.

### Basic Workflow

Recording runs the program with about 2x slowdown while `rr` logs all sources of non-determinism. Once you have a recording, replay drops you into a GDB-like session with extra commands:

```bash
# Record the program (runs ~2x slower)
rr record ./my_program --arg1 --arg2

# Replay - the execution is byte-for-byte identical
rr replay

# Inside the replay GDB session:
(rr) continue            # Run forward
(rr) reverse-continue    # Run backward!
(rr) reverse-step        # Step backward one source line
(rr) reverse-next        # Step backward over function calls

# Example: find where a variable was corrupted
(rr) watch -l my_object.count   # Hardware watchpoint
(rr) reverse-continue           # Run backward to the PREVIOUS write
# -> Stops at the exact instruction that corrupted the value
```

The `reverse-continue` command is the killer feature. You see that `my_object.count` has a wrong value at the crash site. You set a watchpoint on it and run backwards. `rr` stops at the most recent instruction that *wrote* that memory. One command, and you're standing at the line that caused the corruption - no matter how many function calls and loops separated that write from the crash.

### Debugging Race Conditions

Race conditions are the hardest case for conventional debuggers because the bug depends on thread interleaving that changes every run. With `rr`, every run is deterministic - once you've recorded a crash, you can replay it an unlimited number of times with the exact same interleaving:

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

The `when` command is unique to `rr` - it tells you the event number at the current point in the recording. Event numbers are stable across replays, so you can say "the crash happens at event 1234, and I want to investigate what happened at event 1000" and jump directly there with `run 1000`.

### Conditional Reverse Debugging

You can combine watchpoints with conditions just like in regular GDB, but running backwards. This lets you ask very specific questions like "go back to the last moment before this map had more than 1000 entries":

```bash
# Set a conditional watchpoint and go backward
(rr) watch my_map.size()
(rr) condition 1 my_map.size() > 1000
(rr) reverse-continue
# Stops at the moment size exceeded 1000 - going backwards in time
```

This is the kind of query that would be completely impractical with conventional debugging - you'd have to rerun the program with a carefully placed conditional breakpoint and hope the bug reproduces. With `rr` you just ask the question and get the answer.

---

## Self-Assessment

### Q1: What are rr's limitations

`rr` is powerful but not universal. The main constraints are:

- Linux only (uses `perf_event_open`)
- x86/x86-64 only (ARM support experimental)
- Single-machine only (no distributed recording)
- About 2x slowdown during recording
- Doesn't work well inside VMs without hardware perf counter passthrough
- Cannot record GPU operations or kernel-mode code

The Linux/x86 restriction is the most significant in practice. If you're developing on macOS or Windows, or targeting ARM, `rr` is not an option. The VM restriction also catches people off-guard - many CI environments are VMs that don't expose hardware performance counters to guests.

### Q2: How does rr achieve deterministic replay

`rr` records all sources of non-determinism: system call results, signal delivery, thread scheduling decisions, `rdtsc` values, and hardware interrupts. During replay, it intercepts these same events and feeds back the exact recorded values, ensuring the program takes the same execution path. Thread scheduling is the tricky part - `rr` uses hardware performance counters to count instructions executed per thread, which lets it reproduce the exact moments when context switches happened.

The reason this is hard to do in general is that non-determinism comes from many sources. It's not enough to replay system calls - you also need to reproduce timing-dependent behavior, which `rr` handles by controlling the apparent values of time-reading instructions.

### Q3: Compare rr with other time-travel debuggers

| Debugger | Platform | Approach |
| --- | --- | --- |
| rr | Linux | Record/replay via perf counters |
| UDB (UndoDB) | Linux | Commercial, supports ARM |
| WinDbg TTD | Windows | Time Travel Debugging, Microsoft |
| QIRA | Linux | Full system emulation |

`rr` is the only free, open-source option in this table with production-quality reliability. WinDbg TTD is the closest equivalent on Windows and is also excellent, but Windows-only. UDB/UndoDB is commercial and supports a broader set of architectures.

---

## Notes

- `rr` is free and open source - used by Mozilla, Chromium, and many other projects.
- Record once, debug many times: share the recording with teammates.
- `rr chaos` mode randomizes scheduling to find concurrency bugs faster.
- Combine with AddressSanitizer: `rr record ./asan_binary` to get time-travel.
