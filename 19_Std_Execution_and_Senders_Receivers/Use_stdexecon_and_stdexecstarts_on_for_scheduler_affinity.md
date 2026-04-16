# Use stdexec::on and stdexec::starts_on for scheduler affinity

**Category:** std::execution & Senders/Receivers  
**Item:** #708  
**Standard:** C++11  
**Reference:** <https://github.com/NVIDIA/stdexec>  

---

## Topic Overview

Scheduler affinity APIs control which execution context runs each part of a pipeline.

| API | Purpose | Direction |
| --- | --- | --- |
| `starts_on(sched, sender)` | Run entire sender on scheduler | Sets initial context |
| `continues_on(sched)` | Transfer subsequent work to scheduler | Mid-pipeline switch |
| `schedule(sched)` | Create entry-point sender on scheduler | Pipeline start |

---

## Self-Assessment

### Q1: `starts_on` — ensure a pipeline begins on a specific scheduler

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool pool(4);
    auto sched = pool.get_scheduler();

    // A plain sender (runs on whatever context drives it):
    auto computation = stdexec::just(42)
        | stdexec::then([](int x) {
            std::cout << "Computing on thread: "
                      << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    // starts_on: force this sender to run on the pool:
    auto on_pool = stdexec::starts_on(sched, std::move(computation));

    auto [result] = stdexec::sync_wait(std::move(on_pool)).value();
    std::cout << "Result: " << result << '\n';  // 84

    // Without starts_on, sync_wait runs it on its internal run_loop.
    // With starts_on, it runs on the thread pool.
}

```

### Q2: `continues_on` — move continuations to a different scheduler

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>

int main() {
    exec::static_thread_pool compute_pool(4);
    exec::static_thread_pool io_pool(1);
    auto compute = compute_pool.get_scheduler();
    auto io = io_pool.get_scheduler();

    auto pipeline =
        stdexec::schedule(compute)
        | stdexec::then([]() -> int {
            std::cout << "[compute] " << std::this_thread::get_id() << '\n';
            return 42;  // heavy computation
        })
        | stdexec::continues_on(io)   // switch to I/O pool
        | stdexec::then([](int x) -> int {
            std::cout << "[io]      " << std::this_thread::get_id() << '\n';
            // write to disk / send over network
            return x;
        })
        | stdexec::continues_on(compute)  // switch back
        | stdexec::then([](int x) {
            std::cout << "[compute] " << std::this_thread::get_id() << '\n';
            return x * 2;
        });

    auto [result] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Result: " << result << '\n';  // 84
}
// Thread IDs show the context switches between pools

```

### Q3: Full pipeline: compute → I/O → compute

```cpp

#include <stdexec/execution.hpp>
#include <exec/static_thread_pool.hpp>
#include <iostream>
#include <thread>
#include <string>
#include <vector>

int main() {
    exec::static_thread_pool compute_pool(4);
    exec::static_thread_pool io_pool(2);
    auto compute = compute_pool.get_scheduler();
    auto io = io_pool.get_scheduler();

    // Realistic example: image processing pipeline
    auto pipeline =
        // Phase 1: Load image data (I/O pool)
        stdexec::starts_on(io,
            stdexec::just(std::string("/photos/image.raw"))
            | stdexec::then([](const std::string& path) {
                std::cout << "[IO] Loading " << path
                          << " on " << std::this_thread::get_id() << '\n';
                return std::vector<uint8_t>(1024, 128);  // simulated image data
            })
        )
        // Phase 2: Process image (compute pool)
        | stdexec::continues_on(compute)
        | stdexec::then([](std::vector<uint8_t> data) {
            std::cout << "[Compute] Processing "
                      << data.size() << " bytes on "
                      << std::this_thread::get_id() << '\n';
            for (auto& pixel : data) pixel = 255 - pixel;  // invert
            return data;
        })
        // Phase 3: Save result (I/O pool)
        | stdexec::continues_on(io)
        | stdexec::then([](std::vector<uint8_t> data) {
            std::cout << "[IO] Saving " << data.size()
                      << " bytes on " << std::this_thread::get_id() << '\n';
            return data.size();
        });

    auto [bytes] = stdexec::sync_wait(std::move(pipeline)).value();
    std::cout << "Saved " << bytes << " bytes\n";
}
// Output:
// [IO] Loading /photos/image.raw on 14000050
// [Compute] Processing 1024 bytes on 14000001
// [IO] Saving 1024 bytes on 14000051
// Saved 1024 bytes

```

```cpp

Pipeline scheduler flow:

  IO pool          Compute pool        IO pool
┌──────────┐    ┌──────────────┐    ┌──────────┐
│ Load     │───→│ Process      │───→│ Save     │
│ image    │    │ (invert)     │    │ result   │
└──────────┘    └──────────────┘    └──────────┘
  starts_on    continues_on       continues_on

```

---

## Notes

- `starts_on(sched, sender)` sets the initial context; `continues_on(sched)` changes it mid-pipeline.
- `continues_on` was called `transfer` in earlier P2300 drafts.
- `on(sched, sender)` is a shorthand for `starts_on(sched, sender) | continues_on(original_scheduler)`.
- Use I/O schedulers for file/network operations and compute schedulers for CPU-heavy work.
- Scheduler affinity is a zero-cost abstraction — the pipeline compiles to direct task submissions.
