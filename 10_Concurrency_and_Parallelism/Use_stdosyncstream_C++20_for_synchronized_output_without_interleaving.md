# Use `std::osyncstream` (C++20) for Synchronized Output Without Interleaving

**Category:** Concurrency & Parallelism  
**Standard:** C++20 (Header `<syncstream>`)  
**Reference:** [cppreference – std::basic_osyncstream](https://en.cppreference.com/w/cpp/io/basic_osyncstream)  

---

## Topic Overview

When multiple threads write to `std::cout` concurrently, individual `operator<<` calls are thread-safe (no undefined behavior), but **sequences** of insertions from different threads interleave unpredictably. A single logical line like `std::cout << "ID: " << id << "\n";` compiles into multiple calls, and the scheduler can interleave another thread's output between any two of them. The traditional fix—wrapping every print statement in a `std::mutex` lock—is error-prone, composability-hostile, and forces every call site to know about the mutex.

`std::osyncstream` solves this at the library level. It wraps any `std::basic_ostream` and **buffers all output locally** in a `std::basic_syncbuf`. The buffered content is **atomically transferred** to the underlying stream when the `osyncstream` object is destroyed or when `.emit()` is called explicitly. The C++ standard guarantees that characters transferred from different `osyncstream` objects targeting the same stream **do not interleave**—each emission appears as a contiguous block.

| Aspect | `std::mutex` + `cout` | `std::osyncstream` |
| --- | --- | --- |
| Granularity of atomicity | Manual (lock scope) | Automatic (object lifetime) |
| Composability | Poor—callee must know the mutex | Excellent—just pass an `osyncstream` |
| Risk of forgetting the lock | High | None—RAII guarantees flush |
| Performance model | Contention on mutex every insertion | Local buffering, one sync on emit |
| Available since | C++11 | C++20 |

Internally, `std::basic_syncbuf<CharT, Traits, Allocator>` holds a pointer to the wrapped streambuf and an **internal buffer**. On `emit()`, it locks an implementation-defined mutex associated with the wrapped streambuf, copies the buffer contents, and unlocks. Because the lock is held only during the copy—not during the potentially slow formatting—contention is much lower than holding a user-level mutex for the entire print statement.

```cpp

Thread A                         Thread B
─────────────────────────────    ─────────────────────────────
osyncstream out(cout);           osyncstream out(cout);
out << "Hello ";                 out << "World ";
out << "from A\n";               out << "from B\n";
~osyncstream → emit()            ~osyncstream → emit()
   ┌─ lock ──────────┐             ┌─ lock ──────────┐
   │ "Hello from A\n" │             │ "World from B\n" │
   └──────────────────┘             └──────────────────┘
         ▼                                ▼
    cout receives complete blocks—never "Hello World from from A B\n"

```

---

## Self-Assessment

### Q1: What is the output guarantee of `std::osyncstream` and how does it compare to raw `cout` under concurrency

```cpp

// Compile: g++ -std=c++20 -pthread q1_osyncstream_basic.cpp -o q1
#include <iostream>
#include <syncstream>
#include <thread>
#include <vector>

void print_raw(int id) {
    // Multiple operator<< calls: interleaving is possible
    std::cout << "Thread " << id << ": raw output line\n";
}

void print_synced(int id) {
    // Entire sequence is emitted atomically on destruction
    std::osyncstream out(std::cout);
    out << "Thread " << id << ": synced output line\n";
    // ~osyncstream calls emit() here
}

int main() {
    constexpr int N = 8;
    std::vector<std::jthread> threads;

    std::cout << "=== Raw cout (may interleave) ===\n";
    for (int i = 0; i < N; ++i)
        threads.emplace_back(print_raw, i);
    threads.clear();  // join all

    std::cout << "\n=== osyncstream (no interleaving) ===\n";
    for (int i = 0; i < N; ++i)
        threads.emplace_back(print_synced, i);
    threads.clear();

    return 0;
}

```

**Key insight:** Each `osyncstream` temporary buffers all `<<` operations locally. When the object is destroyed at the end of `print_synced`, `emit()` atomically transfers the buffer. Two threads' emissions targeting the same `cout` are serialised—the standard says they "do not interleave."

---

### Q2: How does explicit `emit()` let you control when the atomic flush happens, and what is the state of the `osyncstream` after it

```cpp

// Compile: g++ -std=c++20 -pthread q2_explicit_emit.cpp -o q2
#include <iostream>
#include <syncstream>
#include <thread>
#include <string>

void worker(int id) {
    std::osyncstream out(std::cout);

    // Phase 1: accumulate a progress report
    out << "[Worker " << id << "] Phase 1 complete. ";
    out << "Partial results ready.\n";
    out.emit();  // flush to cout atomically NOW

    // After emit(), the osyncstream is empty but still usable
    // Phase 2: accumulate more output
    out << "[Worker " << id << "] Phase 2 complete. ";
    out << "Final results ready.\n";
    // implicit emit() on destruction
}

int main() {
    std::jthread t1(worker, 1);
    std::jthread t2(worker, 2);
    std::jthread t3(worker, 3);
    return 0;
}

```

**Key insight:** After `emit()`, the internal buffer is cleared and the `osyncstream` can be reused. Each `emit()` is a separate atomic transfer—Phase 1 and Phase 2 outputs are independently non-interleaving blocks. If you need two logically separate messages to never split another thread's message, call `emit()` between them.

---

### Q3: How does `osyncstream` interact with `std::flush`, `std::endl`, and the `emit_on_flush` policy

```cpp

// Compile: g++ -std=c++20 -pthread q3_flush_policy.cpp -o q3
#include <iostream>
#include <syncstream>
#include <thread>
#include <sstream>

void demonstrate_flush_policies() {
    // --- Default: emit_on_flush is true for osyncstream ---
    {
        std::osyncstream out(std::cout);
        out << "Line A with endl" << std::endl;
        // std::endl calls flush() on the osyncstream.
        // Because emit_on_flush is true (default), this triggers emit().
        // "Line A with endl\n" is now in cout.

        out << "Line B after emit";
        // Will be emitted on destruction
    }

    std::cout << "\n--- syncbuf with emit_on_flush(false) ---\n";

    // --- Manual syncbuf control ---
    {
        std::syncbuf sbuf(std::cout.rdbuf());
        sbuf.set_emit_on_sync(false);  // flush won't trigger emit

        std::ostream out(&sbuf);
        out << "This flush won't emit." << std::flush;
        // Nothing appears in cout yet—still in syncbuf's buffer.

        out << " Still buffered.\n";
        sbuf.emit();  // NOW both messages transfer atomically
    }
}

void worker_with_endl(int id) {
    // Common pattern: use temporary osyncstream with endl
    // endl triggers both '\n' AND emit(), giving immediate atomic output
    std::osyncstream(std::cout) << "Worker " << id << " done." << std::endl;
}

int main() {
    demonstrate_flush_policies();

    std::cout << "\n--- Workers with endl ---\n";
    std::jthread t1(worker_with_endl, 1);
    std::jthread t2(worker_with_endl, 2);
    std::jthread t3(worker_with_endl, 3);

    return 0;
}

```

**Key insight:** By default `osyncstream` sets `emit_on_sync(true)` on its underlying `syncbuf`, so `std::flush` / `std::endl` triggers an atomic emit. You can disable this via the underlying `syncbuf` if you want `flush` to be a no-op and control emission manually. The one-liner `std::osyncstream(std::cout) << ... << std::endl;` is the idiomatic pattern: the temporary is destroyed at the semicolon, guaranteeing atomic output.

---

## Notes

- **No interleaving ≠ ordering.** `osyncstream` guarantees each emission is contiguous but does **not** impose a total order on which thread's output appears first. Use external synchronisation (latches, barriers) if ordering matters.
- **Performance:** formatting happens into a thread-local buffer with zero contention. Only the final `emit()` acquires the implementation's internal mutex briefly. This is dramatically better than holding a `std::mutex` around the entire `<<` chain.
- **Nested `osyncstream`:** You can wrap an `osyncstream` in another `osyncstream`. Each level adds its own buffering. This is rarely useful but not UB.
- **File streams:** `osyncstream` works with any `ostream`—`ofstream`, `ostringstream`, etc.—not just `cout`. This is useful for multi-threaded logging to a single file.
- **Temporary idiom:** `std::osyncstream(std::cout) << "msg\n";` is the most concise pattern. The temporary lives until the end of the full expression, so the entire `<<` chain is captured.
- **Allocation:** `syncbuf` may allocate heap memory for its internal buffer. For extremely hot paths, profile to confirm this isn't a bottleneck.
