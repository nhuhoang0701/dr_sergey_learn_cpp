# Use std::string's starts_with, ends_with, and contains (C++20/23)

**Category:** Standard Library — Containers  
**Item:** #204  
**Standard:** C++20 (`starts_with`, `ends_with`), C++23 (`contains`)  
**Reference:** <https://en.cppreference.com/w/cpp/string/basic_string/starts_with>  

---

## Topic Overview

C++20 added `starts_with()` and `ends_with()` to `std::string` and `std::string_view`. C++23 added `contains()`. These member functions replace awkward, error-prone idioms with clear, readable checks.

### Before vs After

| Check | Old way (pre-C++20) | New way |
| --- | --- | --- |
| Starts with prefix | `s.substr(0, prefix.size()) == prefix` | `s.starts_with(prefix)` |
| | `s.find(prefix) == 0` | |
| | `s.compare(0, prefix.size(), prefix) == 0` | |
| Ends with suffix | `s.size() >= suffix.size() && s.compare(s.size()-suffix.size(), suffix.size(), suffix) == 0` | `s.ends_with(suffix)` |
| Contains substring | `s.find(sub) != std::string::npos` | `s.contains(sub)` (C++23) |

### API Summary

```cpp

#include <string>
#include <string_view>

std::string s = "Hello, World!";

// === starts_with (C++20) ===
s.starts_with("Hello");    // true
s.starts_with("World");    // false
s.starts_with('H');        // true  (single char overload)
s.starts_with("");         // true  (empty prefix always matches)

// === ends_with (C++20) ===
s.ends_with("World!");     // true
s.ends_with("Hello");      // false
s.ends_with('!');          // true

// === contains (C++23) ===
s.contains("lo, W");       // true
s.contains("xyz");         // false
s.contains(',');           // true

// All three also work on std::string_view
std::string_view sv = s;
sv.starts_with("Hello");   // true
sv.contains("World");      // true

```

### Overloads

Each function accepts three overloads:

1. `starts_with(string_view sv)` — check against a string/view
2. `starts_with(charT c)` — check against a single character
3. `starts_with(const charT* s)` — check against a C-string

---

## Self-Assessment

### Q1: Replace a substr(0, prefix.size()) == prefix check with str.starts_with(prefix)

```cpp

#include <iostream>
#include <string>
#include <string_view>
#include <vector>

// === Old way: multiple approaches, all error-prone ===
bool starts_with_old_1(const std::string& s, const std::string& prefix) {
    return s.substr(0, prefix.size()) == prefix;
    // Problem: creates a temporary string (allocation!)
}

bool starts_with_old_2(const std::string& s, const std::string& prefix) {
    return s.find(prefix) == 0;
    // Problem: scans ENTIRE string looking for prefix anywhere, then checks pos==0
    // O(n*m) when O(m) would suffice
}

bool starts_with_old_3(const std::string& s, const std::string& prefix) {
    return s.size() >= prefix.size() &&
           s.compare(0, prefix.size(), prefix) == 0;
    // Correct and efficient, but verbose and easy to get wrong
}

int main() {
    std::vector<std::string> paths = {
        "/usr/local/bin/gcc",
        "/usr/share/doc/readme.txt",
        "/home/user/.config/settings",
        "/tmp/scratch_001",
        "/usr/local/lib/libfoo.so"
    };

    std::string prefix = "/usr/local/";

    // === Old way ===
    std::cout << "Old way (substr):\n";
    for (const auto& p : paths) {
        if (p.substr(0, prefix.size()) == prefix)
            std::cout << "  " << p << "\n";
    }

    // === New way (C++20) ===
    std::cout << "\nNew way (starts_with):\n";
    for (const auto& p : paths) {
        if (p.starts_with(prefix))
            std::cout << "  " << p << "\n";
    }
    // Both output:
    //   /usr/local/bin/gcc
    //   /usr/local/lib/libfoo.so

    // === ends_with for file extensions ===
    std::cout << "\n.so files (ends_with):\n";
    for (const auto& p : paths) {
        if (p.ends_with(".so"))
            std::cout << "  " << p << "\n";
    }
    //   /usr/local/lib/libfoo.so

    // === Edge cases ===
    std::string s = "Hello";
    std::cout << std::boolalpha;
    std::cout << "\nEdge cases:\n";
    std::cout << "  starts_with(\"\"):     " << s.starts_with("") << "\n";       // true
    std::cout << "  starts_with(\"Hello\"): " << s.starts_with("Hello") << "\n"; // true (exact match)
    std::cout << "  starts_with(\"HelloX\"): " << s.starts_with("HelloX") << "\n"; // false (prefix longer)
    std::cout << "  ends_with(\"\"):       " << s.ends_with("") << "\n";         // true

    return 0;
}

```

**How it works:**

- `starts_with(prefix)` compares the first `prefix.size()` characters. Returns `false` immediately if the string is shorter than the prefix.
- **No allocation:** Unlike `substr()`, `starts_with` doesn't create a temporary string.
- **No over-scanning:** Unlike `find(prefix) == 0`, it only checks position 0.
- Complexity: O(min(size(), prefix.size())) — compares only as many characters as needed.

### Q2: Use contains() for a simple substring search and show its readability advantage

```cpp

#include <iostream>
#include <string>
#include <vector>

int main() {
    // === C++23 contains() ===
    std::vector<std::string> log_lines = {
        "[INFO] Server started on port 8080",
        "[ERROR] Connection refused: timeout",
        "[WARN] Disk usage at 85%",
        "[ERROR] Out of memory",
        "[INFO] Request processed in 42ms",
        "[DEBUG] Cache hit ratio: 0.95"
    };

    // === Old way: find() != npos ===
    std::cout << "Errors (old way):\n";
    for (const auto& line : log_lines) {
        if (line.find("[ERROR]") != std::string::npos)
            std::cout << "  " << line << "\n";
    }

    // === New way: contains() (C++23) ===
    std::cout << "\nErrors (C++23 contains):\n";
    for (const auto& line : log_lines) {
        if (line.contains("[ERROR]"))
            std::cout << "  " << line << "\n";
    }
    // Both output:
    //   [ERROR] Connection refused: timeout
    //   [ERROR] Out of memory

    // === Readability comparison ===
    std::string url = "https://example.com/api/v2/users?active=true";

    // Old: double negation, magic constant
    if (url.find("api/v2") != std::string::npos) { /* ... */ }
    if (url.find('?') != std::string::npos) { /* ... */ }

    // New: reads like English
    if (url.contains("api/v2")) { /* ... */ }
    if (url.contains('?')) { /* ... */ }

    // === Combining all three ===
    std::cout << "\nURL analysis:\n";
    std::cout << std::boolalpha;
    std::cout << "  HTTPS:      " << url.starts_with("https://") << "\n";  // true
    std::cout << "  Has query:  " << url.contains('?') << "\n";            // true
    std::cout << "  Is JSON:    " << url.ends_with(".json") << "\n";       // false
    std::cout << "  Has API v2: " << url.contains("api/v2") << "\n";      // true

    return 0;
}

```

**How it works:**

- `contains(x)` is equivalent to `find(x) != npos` — same semantics, but far more readable.
- Available for both `std::string` and `std::string_view`.
- The readability gain is most obvious in conditions: `if (s.contains("error"))` reads like natural English compared to `if (s.find("error") != std::string::npos)`.
- `contains()` is C++23-only. For C++20, use `find() != npos` or write a helper.

### Q3: Show performance considerations: contains is O(n*m) — use find or Boyer-Moore for large text

```cpp

#include <iostream>
#include <string>
#include <algorithm>
#include <functional>
#include <chrono>

int main() {
    // === Performance characteristics ===
    //
    // starts_with(prefix):  O(m)     — compare m chars at position 0
    // ends_with(suffix):    O(m)     — compare m chars at end
    // contains(pattern):    O(n*m)   — find() internally (naive search)
    //   where n = string length, m = pattern length
    //
    // For SHORT strings and patterns, this is perfectly fine.
    // For LARGE text with LONG patterns, consider alternatives.

    // === Generating test data ===
    std::string large_text(1'000'000, 'a');  // 1M chars
    large_text += "NEEDLE";                   // pattern at the very end

    std::string pattern = "NEEDLE";

    // === Method 1: contains / find (naive) ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        bool found = large_text.contains(pattern);
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "contains():     found=" << found << "  time=" << us << " us\n";
    }

    // === Method 2: std::search with default (may use better algorithm) ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              pattern.begin(), pattern.end());
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "std::search:    found=" << found << "  time=" << us << " us\n";
    }

    // === Method 3: Boyer-Moore (C++17) — O(n/m) best case ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              std::boyer_moore_searcher(pattern.begin(), pattern.end()));
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "Boyer-Moore:    found=" << found << "  time=" << us << " us\n";
    }

    // === Method 4: Boyer-Moore-Horspool (C++17) — simpler, often faster ===
    {
        auto t0 = std::chrono::high_resolution_clock::now();
        auto it = std::search(large_text.begin(), large_text.end(),
                              std::boyer_moore_horspool_searcher(pattern.begin(), pattern.end()));
        bool found = (it != large_text.end());
        auto t1 = std::chrono::high_resolution_clock::now();
        auto us = std::chrono::duration_cast<std::chrono::microseconds>(t1 - t0).count();
        std::cout << "BM-Horspool:    found=" << found << "  time=" << us << " us\n";
    }

    // === When to use what ===
    std::cout << "\n=== Guidelines ===\n";
    std::cout << "Use starts_with/ends_with: Always. O(m), no alternatives needed.\n";
    std::cout << "Use contains/find:         For short strings (< few KB). Simple, fast enough.\n";
    std::cout << "Use Boyer-Moore:           For large text (MB+) with long patterns.\n";
    std::cout << "Use regex:                 Only when you need pattern matching, not substring search.\n";

    // === Complexity summary ===
    // | Algorithm            | Preprocessing | Search      | Best case   |
    // |---------------------|---------------|-------------|-------------|
    // | Naive (find/contains)| O(1)         | O(n*m)      | O(n)        |
    // | Boyer-Moore          | O(m + σ)     | O(n*m) worst| O(n/m)      |
    // | Boyer-Moore-Horspool | O(σ)         | O(n*m) worst| O(n/m)      |
    // | KMP                  | O(m)         | O(n)        | O(n)        |
    //
    // σ = alphabet size, n = text length, m = pattern length

    return 0;
}

```

**How it works:**

- `contains()` / `find()` use the naive substring search algorithm: O(n×m) worst case, but typically O(n) for natural text.
- **Boyer-Moore** (C++17): Preprocesses the pattern to skip large portions of text. Best case O(n/m) — the longer the pattern, the faster it gets.
- **Boyer-Moore-Horspool**: Simplified Boyer-Moore with only the bad-character heuristic. Slightly less preprocessing, often comparable speed.
- For `starts_with` and `ends_with`, there's no performance concern — they're always O(m) regardless of string length.
- **Rule of thumb:** Use `contains()` for strings under a few KB. For document/log searching (MB+), use `std::search` with a Boyer-Moore searcher.

---

## Notes

- **`starts_with` / `ends_with`** are C++20. **`contains`** is C++23.
- All three are `const noexcept` — no exceptions, no allocations, safe to use anywhere.
- Available on both `std::string` and `std::string_view` (and `std::basic_string<CharT>`).
- **Empty patterns:** `starts_with("")` and `ends_with("")` and `contains("")` all return `true` — consistent with mathematical definitions.
- **Case-insensitive:** None of these functions support case-insensitive comparison. For that, convert both strings to lowercase first, or use `std::ranges::equal` with a custom comparator.
- **`string_view` advantage:** These functions take `string_view`, so you can pass `const char*`, `string`, or `string_view` — no temporary allocations.
- **Compiler support:** `starts_with`/`ends_with` — GCC 11+, Clang 12+, MSVC 19.30+. `contains` — GCC 13+, Clang 17+, MSVC 19.34+.
