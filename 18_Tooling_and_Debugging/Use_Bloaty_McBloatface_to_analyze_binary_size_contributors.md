# Use Bloaty McBloatface to analyze binary size contributors

**Category:** Tooling & Debugging  
**Item:** #798  
**Reference:** <https://github.com/google/bloaty>  

---

## Topic Overview

Bloaty can drill down to individual **symbols** and **compilation units** to pinpoint template instantiation bloat, the #1 cause of large C++ binaries.

```bash

# Show top symbols:
bloaty -d symbols -n 20 ./myapp

# Show top compilation units:
bloaty -d compileunits -n 10 ./myapp

# Two-level: symbols within compilation units:
bloaty -d compileunits,symbols -n 10 ./myapp

```

---

## Self-Assessment

### Q1: Identify which symbols consume the most space

```bash

bloaty -d symbols -n 15 ./myapp

```

Typical output:

```text

    FILE SIZE        VM SIZE
 --------------  --------------
  15.3%   180Ki  15.3%   180Ki   std::__cxx11::basic_string<char, ...>
   9.8%   115Ki   9.8%   115Ki   std::vector<Widget, ...>::_M_realloc_insert
   7.2%    85Ki   7.2%    85Ki   std::sort<__gnu_cxx::__normal_iterator<...>>
   5.1%    60Ki   5.1%    60Ki   nlohmann::json::parse(...)
   4.3%    51Ki   4.3%    51Ki   MyApp::processData()

```

Key patterns to look for:

- **STL symbols** (string, vector, map): template instantiations for each type
- **Third-party libraries**: JSON, logging, etc.
- **Exception handling**: `__cxa_throw`, `.eh_frame`, `.gcc_except_table`

### Q2: Find and eliminate template instantiation bloat

```cpp

// BEFORE: Each type creates a full copy of the entire function
// -> log<int>, log<double>, log<string>, log<Widget> = 4 copies!

#include <iostream>
#include <string>
#include <typeinfo>

// BAD: full template — entire body duplicated per instantiation
template<typename T>
void log_value_bad(const std::string& label, const T& value) {
    std::cout << "[" << __FILE__ << ":" << __LINE__ << "] "
              << label << " = ";
    // 50 lines of formatting, buffering, flushing...
    std::cout << value << '\n';
}

// GOOD: type-erased core + thin template wrapper
void log_impl(const std::string& label, const std::string& str_value) {
    std::cout << label << " = " << str_value << '\n';
    // All 50 lines of formatting only exist ONCE
}

template<typename T>
void log_value_good(const std::string& label, const T& value) {
    // Thin wrapper: only the conversion is templated
    if constexpr (std::is_same_v<T, std::string>) {
        log_impl(label, value);
    } else {
        log_impl(label, std::to_string(value));
    }
}

int main() {
    log_value_good("count", 42);
    log_value_good("ratio", 3.14);
    log_value_good("name", std::string("Alice"));
}
// Expected output:
// count = 42
// ratio = 3.140000
// name = Alice

```

Verify with Bloaty:

```bash

# Before (full template):
bloaty -d symbols ./before | grep log_value
#   log_value<int>    2.1Ki
#   log_value<double> 2.1Ki
#   log_value<string> 2.3Ki

# After (thin wrapper + shared impl):
bloaty -d symbols ./after | grep log
#   log_impl          2.1Ki  (shared)
#   log_value<int>    0.1Ki  (tiny wrapper)
#   log_value<double> 0.1Ki

```

### Q3: `std::function` vs template parameter binary size comparison

```cpp

#include <iostream>
#include <functional>
#include <vector>

// Version A: std::function (type-erased, ONE instantiation)
void process_func(const std::vector<int>& data,
                  std::function<int(int)> transform) {
    for (auto x : data)
        std::cout << transform(x) << ' ';
    std::cout << '\n';
}

// Version B: template (separate instantiation per callable type)
template<typename F>
void process_tmpl(const std::vector<int>& data, F transform) {
    for (auto x : data)
        std::cout << transform(x) << ' ';
    std::cout << '\n';
}

int main() {
    std::vector<int> v = {1, 2, 3, 4, 5};

    // std::function: one instantiation, but ~400 bytes runtime overhead
    process_func(v, [](int x) { return x * 2; });
    process_func(v, [](int x) { return x + 10; });

    // template: two instantiations, but inline-able (faster)
    process_tmpl(v, [](int x) { return x * 2; });
    process_tmpl(v, [](int x) { return x + 10; });
}
// Expected output:
// 2 4 6 8 10
// 11 12 13 14 15
// 2 4 6 8 10
// 11 12 13 14 15

```

| Approach | Code Size | Runtime Speed | Use When |
| --- | --- | --- | --- |
| `std::function` | Smaller (1 instantiation) | Slower (indirect call) | Binary size matters, many callers |
| Template | Larger (N instantiations) | Faster (inlined) | Hot path, few distinct callers |

---

## Notes

- Bloaty's diff mode (`bloaty new -- old`) is essential for tracking size regressions.
- Common bloat sources: `<iostream>` (~100KB), `<regex>` (~300KB), exception tables.
- Use `extern template` to prevent unwanted instantiations in headers.
- `-ffunction-sections -fdata-sections -Wl,--gc-sections` removes unused functions.
- Consider thin template wrappers around type-erased implementations.
