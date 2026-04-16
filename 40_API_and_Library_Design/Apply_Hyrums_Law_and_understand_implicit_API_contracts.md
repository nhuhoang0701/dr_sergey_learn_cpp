# Apply Hyrum's Law and understand implicit API contracts

**Category:** API & Library Design  
**Standard:** C++17  
**Reference:** <https://www.hyrumslaw.com/>  

---

## Topic Overview

**Hyrum's Law**: "With a sufficient number of users of an API, all observable behaviors of your system will be depended on by somebody." This means even undocumented behavior becomes part of your API.

### Practical Implications

```cpp

#include <map>
#include <string>
#include <iostream>

// std::map iterates in sorted order — this is documented.
// But users also depend on:
// - The exact formatting of error messages (even though it's unspecified)
// - The iteration order of std::unordered_map (implementation-defined)
// - The exact sizeof (which differs between compilers/platforms)
// - Performance characteristics (O(1) amortized is not O(1) worst case)

// Example: changing message text broke real users
void before_v2() {
    // v1: throws with message "file not found: config.yaml"
    // Users parsed this string to extract the filename!
}

void after_v2() {
    // v2: changed to "cannot open file: config.yaml"
    // BROKE users who did: msg.substr(msg.find(": ") + 2)
}

// LESSON: error messages, whitespace in output, iteration order
// are all implicit API contracts.

```

### Defensive API Design

```cpp

#include <string>
#include <optional>

// BAD: leaking implementation details
class BadCache {
public:
    // Users can see the internal map and depend on its properties
    const std::map<std::string, std::string>& entries() const { return map_; }
private:
    std::map<std::string, std::string> map_;
};

// GOOD: minimal interface — internal representation is hidden
class GoodCache {
public:
    std::optional<std::string> get(const std::string& key) const;
    void put(const std::string& key, std::string value);
    size_t size() const;
    // No access to internal container — free to change implementation
private:
    std::unordered_map<std::string, std::string> map_; // Could be anything
};

```

---

## Self-Assessment

### Q1: Give examples of implicit API contracts in the C++ standard library

- `std::vector` elements are contiguous in memory (documented, but users also depend on `&v[0]` being a valid C array pointer)
- `std::string` is null-terminated (observable, depended on by C interop code)
- `std::sort` is not stable (but some users rely on specific behavior of introsort)
- Move operations leave objects in a "valid but unspecified state" — but users depend on moved-from `string` being empty

### Q2: How to minimize implicit contract surface

- Export minimal interface; hide implementation with Pimpl
- Don't let internal containers leak through the API
- Use opaque error types instead of string messages
- Document what IS guaranteed and explicitly mark everything else as "may change"

### Q3: What are "chesterton's fence" changes in API evolution

Before removing or changing any behavior, understand WHY it exists (even if undocumented). Hyrum's Law means someone depends on it. The fix is: add the new behavior alongside the old, deprecate the old, remove after a migration period.

---

## Notes

- Hyrum's Law applies to all APIs: C++, REST, CLI tools, config file formats.
- Fewer observable behaviors = fewer implicit contracts = easier evolution.
- Semantic versioning helps: breaking changes increment the major version.
- "Don't expose what you don't want to maintain forever."
