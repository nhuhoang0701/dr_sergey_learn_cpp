# Use Tracy, Orbit, or Superluminal for real-time profiling

**Category:** Tooling & Debugging  
**Item:** #180  
**Reference:** <https://github.com/wolfpld/tracy>  

---

## Topic Overview

Real-time profilers provide timeline visualization of CPU/GPU activity with live streaming.

| Tool | Platform | Type | License |
| --- | --- | --- | --- |
| **Tracy** | Cross-platform | Instrumentation + sampling | MIT |
| **Orbit** | Linux/Windows | Sampling + dynamic instrumentation | BSD |
| **Superluminal** | Windows | Sampling + context switches | Commercial |
| **perf** | Linux | Sampling | GPL |
| **Intel VTune** | Cross-platform | HW counters + sampling | Free |

---

## Self-Assessment

### Q1: Instrument with Tracy zones and visualize

```cpp

// tracy_demo.cpp
// Link with Tracy: add tracy as a dependency or include TracyClient.cpp
#include <tracy/Tracy.hpp>
#include <vector>
#include <algorithm>
#include <random>
#include <thread>
#include <chrono>

void parse_data(const std::vector<int>& data) {
    ZoneScoped;  // Tracy: auto-named zone from function name
    // Parsing work...
    int sum = 0;
    for (int x : data) sum += x;
    (void)sum;
}

void sort_data(std::vector<int>& data) {
    ZoneScopedN("CustomSort");  // Tracy: named zone
    std::sort(data.begin(), data.end());
}

void process_frame(int frame_num) {
    ZoneScoped;
    ZoneValue(frame_num);  // attach frame number to zone

    std::vector<int> data(100000);
    {
        ZoneScopedN("Generate");  // nested zone
        std::mt19937 rng(frame_num);
        std::generate(data.begin(), data.end(), rng);
    }

    parse_data(data);
    sort_data(data);

    FrameMark;  // Tracy: mark end of frame
}

int main() {
    for (int i = 0; i < 100; ++i) {
        process_frame(i);
    }
}

```

```cmake

# CMake integration:
find_package(Tracy CONFIG REQUIRED)
target_link_libraries(myapp PRIVATE Tracy::TracyClient)
target_compile_definitions(myapp PRIVATE TRACY_ENABLE)

# Or via vcpkg:
# vcpkg install tracy

```

```cpp

Tracy profiler UI shows:
┌─────────────────────────────────────────────────┐
│ Timeline (horizontal = time)                    │
│                                                 │
│ Thread 1: █process_frame██████████████████████│
│           │Generate│parse  │ CustomSort       │  │
│                                                 │
│ Frame: 42  Duration: 2.3ms                      │
│ Zones: 4  CPU: 98%                              │
└─────────────────────────────────────────────────┘

```

### Q2: Detect lock contention with a profiler

```cpp

// contention.cpp
#include <tracy/Tracy.hpp>
#include <mutex>
#include <thread>
#include <vector>

std::mutex global_mutex;
std::vector<int> shared_data;

void worker(int id) {
    ZoneScoped;
    for (int i = 0; i < 100000; ++i) {
        {
            // Tracy tracks lock contention:
            std::lock_guard lock(global_mutex);
            LockMark(global_mutex);  // Tracy: mark lock acquisition
            shared_data.push_back(id * 100000 + i);
        }
    }
}

int main() {
    std::vector<std::thread> threads;
    for (int i = 0; i < 8; ++i)
        threads.emplace_back(worker, i);
    for (auto& t : threads)
        t.join();
}

// Tracy shows:
// - Lock wait time per thread (red bars = contention)
// - Which thread held the lock when others were waiting
// - Total lock contention as percentage of wall time
//
// Fix: shard the data, use lock-free queue, reduce critical section

```

```cpp

// Tracy lock tracking (advanced):
#include <tracy/Tracy.hpp>

// Annotate your mutex:
TracyLockable(std::mutex, my_mutex);  // wraps mutex with tracking

void worker() {
    std::lock_guard<LockableBase(std::mutex)> lock(my_mutex);
    // Tracy now tracks: wait time, hold time, contention count
}

```

### Q3: Sampling vs instrumentation profilers

| Aspect | Sampling (perf, VTune) | Instrumentation (Tracy, Orbit) |
| --- | --- | --- |
| **How** | Periodically samples stack | Code explicitly marks zones |
| **Overhead** | Low (~1-5%) | Variable (per instrumentation point) |
| **Precision** | Statistical (more samples = better) | Exact timing for each zone |
| **Setup** | No code changes | Must add macros/annotations |
| **Timeline** | Post-mortem analysis | Real-time live streaming (Tracy) |
| **Visibility** | All functions (even 3rd party) | Only instrumented functions |
| **Best for** | Finding unknown hotspots | Understanding known hot paths |
| **Frame-based** | Not natively | Yes (Tracy FrameMark) |

```cpp

Typical workflow:

1. Use sampling profiler (perf) to FIND the hotspot
2. Instrument the hotspot with Tracy for DETAILED analysis
3. Optimize based on Tracy's precise timing
4. Verify with perf that overall time decreased

```

---

## Notes

- Tracy has near-zero overhead for non-active zones (~1ns per zone entry).
- Tracy supports GPU profiling (Vulkan, OpenGL, D3D12).
- Orbit (Google) can attach to running processes without recompilation.
- Superluminal excels at context switch and thread migration analysis.
- Use `TracyAllocS`/`TracyFreeS` to track memory allocations in timeline.
