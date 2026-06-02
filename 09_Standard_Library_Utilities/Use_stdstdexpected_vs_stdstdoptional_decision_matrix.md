# Use std::expected vs std::optional decision matrix

**Category:** Standard Library Utilities  
**Standard:** C++23  
**Reference:** <https://en.cppreference.com/w/cpp/utility/expected>  

---

## Topic Overview

Both `std::optional` and `std::expected` represent "maybe a value," but they answer slightly different questions. `optional` says "there might not be a value, and that's fine - no explanation needed." `expected` says "there might not be a value, and if so here is exactly what went wrong." Picking the right one makes your API's intent crystal clear to the caller.

### Decision Matrix

| Question | `optional<T>` | `expected<T, E>` |
| --- | --- | --- |
| Is there error information? | No (just "empty") | Yes (carries error E) |
| Multiple failure reasons? | No | Yes |
| Used for "not found" / "no value"? | Yes | Overkill |
| Monadic chaining? | C++23 yes | C++23 yes |
| Zero overhead? | Yes (no error) | Yes (union of T and E) |

If the table feels like a lot, the rule of thumb is: use `optional` when absent is normal and boring (like a missing map entry), use `expected` when failure has a reason the caller must handle (like a file operation).

### When to Use optional

Use `optional` when absence is simply a normal outcome - nothing went wrong, there's just no value to return. A database lookup that finds nothing is not an error; it's an expected outcome.

```cpp
#include <optional>
#include <string>
#include <map>

// "Not found" - there's nothing wrong, just no value
std::optional<std::string> find_user(int id) {
    static std::map<int, std::string> db = {{1, "Alice"}, {2, "Bob"}};
    if (auto it = db.find(id); it != db.end())
        return it->second;
    return std::nullopt;  // Not an error - just absent
}
```

The caller gets back either a name or `std::nullopt`, and they don't need to ask why - there's only one reason: the id wasn't there.

### When to Use expected

Use `expected` when an operation can fail in multiple distinct ways that the caller needs to distinguish and respond to differently. Here, `FileError` gives the caller the information to decide what to do next.

```cpp
#include <expected>
#include <string>
#include <fstream>

enum class FileError { NotFound, PermissionDenied, TooLarge, Corrupted };

// Multiple failure modes that callers need to distinguish
std::expected<std::string, FileError> read_config(const std::string& path) {
    std::ifstream f(path);
    if (!f) return std::unexpected(FileError::NotFound);

    std::string content;
    // ... read and validate
    if (content.size() > 1'000'000)
        return std::unexpected(FileError::TooLarge);

    return content;
}

void handle() {
    auto result = read_config("app.cfg");
    if (!result) {
        switch (result.error()) {
            case FileError::NotFound: /* ... */ break;
            case FileError::PermissionDenied: /* ... */ break;
            default: break;
        }
    }
}
```

Notice that the caller can inspect `result.error()` and branch on exactly what went wrong - something `optional` simply can't express.

---

## Self-Assessment

### Q1: Can optional and expected be composed

Yes, both have `and_then`, `transform`, and `or_else` in C++23. This lets you chain operations without manually checking each step:

```cpp
auto result = find_user(id)
    .transform([](const auto& name) { return name.size(); })
    .value_or(0);
```

The `.transform` step is skipped entirely if `find_user` returned `nullopt`, and `value_or(0)` gives you a safe default at the end. Clean, no `if` checks in sight.

### Q2: What about performance - is expected larger than optional

`expected<T, E>` = `sizeof(max(T, E)) + discriminant`. `optional<T>` = `sizeof(T) + bool`. If your error type `E` is smaller than `T`, they're the same size. If `E` is large, `expected` is larger.

### Q3: When to use exceptions instead

Use exceptions for truly exceptional situations (can't construct object, out of memory). Use `expected` for anticipated failures in the normal flow (file not found, parse error). Use `optional` for "value or nothing" with no error information.

---

## Notes

- `optional` = maybe a value. `expected` = value or error. Exceptions = unexpected failure.
- Both are monadic in C++23 - composable with `and_then`/`transform`.
- `std::expected<void, E>` is valid for operations that succeed or fail with no return value.
- Prefer `expected` over out-parameters and error codes for new APIs.
