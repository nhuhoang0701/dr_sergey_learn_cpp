# Use -ftime-trace (Clang) to identify compilation bottlenecks

**Category:** Tooling & Debugging  
**Item:** #417  
**Reference:** <https://aras-p.info/blog/2019/01/16/time-trace-timeline-flame-chart-profiler-for-Clang/>  

---

## Topic Overview

`-ftime-trace` makes Clang emit a JSON file showing **exactly** where compilation time is spent: parsing, template instantiation, code generation, header inclusion.

```bash

clang++ -ftime-trace -c myfile.cpp
  => produces myfile.json (Chrome Trace Format)
  => open in chrome://tracing or https://ui.perfetto.dev

```

```cpp

Timeline view (chrome://tracing):
┌──────────────────────────────────────────────────┐
│ Source       │ <vector>  │ <map>     │ myfile.h  │
├──────────────┼───────────┼───────────┼───────────┤
│ Instantiate  │ vector<Widget>            │ map<...>  │
├──────────────┼──────────────────────────┼───────────┤
│ CodeGen      │ main()   │ Widget::foo()              │
└──────────────┴──────────┴─────────────────────────┘
  0ms       100ms      200ms      300ms       400ms

```

---

## Self-Assessment

### Q1: Compile with `-ftime-trace` and find the slowest instantiation

```bash

# Step 1: Compile with -ftime-trace
clang++ -std=c++20 -ftime-trace -c heavy_templates.cpp

# Step 2: This produces heavy_templates.json
# Open in chrome://tracing or https://ui.perfetto.dev

# Step 3: Look for the "InstantiateFunction" or "InstantiateClass" events
# Sort by duration (wall time) to find the slowest

```

Example C++ file that causes slow instantiation:

```cpp

// heavy_templates.cpp
#include <vector>
#include <map>
#include <string>
#include <algorithm>
#include <numeric>
#include <functional>

template<typename T>
struct HeavyWidget {
    std::vector<T> data;
    std::map<std::string, T> index;

    void process() {
        std::sort(data.begin(), data.end());
        auto sum = std::accumulate(data.begin(), data.end(), T{});
    }
};

// Each instantiation pulls in sort, accumulate, map for that type
template struct HeavyWidget<int>;
template struct HeavyWidget<double>;
template struct HeavyWidget<std::string>;

int main() {
    HeavyWidget<int> w;
    w.data = {3, 1, 2};
    w.process();
}

```

In the trace JSON, look for:

- `"name": "InstantiateClass"` with `"detail": "HeavyWidget<std::string>"` — likely the slowest
- `"name": "Source"` entries to see which `#include` takes longest

### Q2: Replace a heavy header with a forward declaration

```cpp

// ========= BEFORE: includes entire <map> just for a pointer =========
// widget.h
#include <map>       // pulls in ~50K lines!
#include <string>

class Widget {
    std::map<std::string, int>* index_;  // only uses pointer
public:
    void lookup(const std::string& key);
};

// ========= AFTER: forward declare, include only in .cpp =========
// widget.h
#include <string>
#include <memory>

// Forward declaration not possible for std:: types, so use Pimpl:
class Widget {
    struct Impl;
    std::unique_ptr<Impl> impl_;
public:
    Widget();
    ~Widget();
    void lookup(const std::string& key);
};

// widget.cpp
#include "widget.h"
#include <map>  // heavy include only in ONE translation unit

struct Widget::Impl {
    std::map<std::string, int> index;
};

Widget::Widget() : impl_(std::make_unique<Impl>()) {}
Widget::~Widget() = default;

void Widget::lookup(const std::string& key) {
    auto it = impl_->index.find(key);
    // ...
}

```

Check impact with `-ftime-trace`:

```bash

# Before: widget.h includes <map>
clang++ -ftime-trace -c client.cpp  # client.cpp includes widget.h
# -> trace shows ~80ms for <map> parsing

# After: widget.h uses Pimpl, no <map>
clang++ -ftime-trace -c client.cpp
# -> trace shows <map> parsing is GONE from this TU

```

### Q3: PCH impact comparison

```bash

# Step 1: Create a precompiled header
# pch.h:
#   #include <vector>
#   #include <map>
#   #include <string>
#   #include <algorithm>
#   #include <iostream>

# Compile the PCH:
clang++ -std=c++20 -ftime-trace -x c++-header pch.h -o pch.h.pch
# -> pch.json shows: Source <vector> 20ms, <map> 30ms, etc

# Step 2: Compile WITHOUT PCH
clang++ -std=c++20 -ftime-trace -c main.cpp
# -> main.json: Source total ~150ms (re-parses all headers)

# Step 3: Compile WITH PCH
clang++ -std=c++20 -ftime-trace -include-pch pch.h.pch -c main.cpp
# -> main.json: Source total ~10ms (PCH loaded in one shot)

```

| Metric | Without PCH | With PCH |
| --- | --- | --- |
| Header parsing | ~150ms | ~10ms |
| Template instantiation | Same | Same |
| Total compile time | ~250ms | ~110ms |

**CMake PCH setup:**

```cmake

target_precompile_headers(myapp PRIVATE
    <vector> <map> <string> <algorithm> <iostream>
)

```

---

## Notes

- `-ftime-trace` is **Clang-only** (GCC has `-ftime-report` but less detailed).
- Use `-ftime-trace-granularity=N` to control minimum event duration (default 500µs).
- Common bottlenecks: `<regex>`, `<iostream>`, `<map>`, `<variant>`, Boost headers.
- Forward declarations and Pimpl help the most for widely-included headers.
- PCH helps when many TUs include the same set of heavy headers.
