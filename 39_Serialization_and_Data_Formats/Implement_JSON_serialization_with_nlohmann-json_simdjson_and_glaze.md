# Implement JSON serialization with nlohmann-json, simdjson, and glaze

**Category:** Serialization & Data Formats  
**Standard:** C++17/20  
**Reference:** <https://github.com/nlohmann/json> · <https://simdjson.org> · <https://github.com/stephenberry/glaze>  

---

## Topic Overview

C++ has no built-in JSON support. Three libraries cover the spectrum from ease-of-use to maximum performance.

### Comparison

| Library | Style | Parse Speed | Ease of Use | Reflection | Allocation |
| --- | --- | --- | --- | --- | --- |
| nlohmann/json | DOM, SAX | ~400 MB/s | Excellent | No (macros) | Yes |
| simdjson | On-demand | ~4 GB/s | Good | No | Minimal |
| glaze | Compile-time | ~2 GB/s | Good | Yes (C++20) | Minimal |

### nlohmann/json — The Ergonomic Choice

```cpp

#include <nlohmann/json.hpp>
#include <iostream>
#include <string>
#include <vector>

using json = nlohmann::json;

struct Person {
    std::string name;
    int age;
    std::vector<std::string> hobbies;
};

// Macro-based serialization (non-intrusive)
NLOHMANN_DEFINE_TYPE_NON_INTRUSIVE(Person, name, age, hobbies)

int main() {
    // Serialize
    Person alice{"Alice", 30, {"coding", "hiking"}};
    json j = alice;
    std::cout << j.dump(2) << "\n";
    // {
    //   "age": 30,
    //   "hobbies": ["coding", "hiking"],
    //   "name": "Alice"
    // }

    // Deserialize
    auto text = R"({"name":"Bob","age":25,"hobbies":["reading"]})";
    auto bob = json::parse(text).get<Person>();
    std::cout << bob.name << " is " << bob.age << "\n";

    // Dynamic access
    json config = json::parse(R"({"debug": true, "port": 8080})");
    bool debug = config.value("debug", false);
    int port = config.value("port", 3000);
    std::cout << "Port: " << port << ", debug: " << debug << "\n";
}

```

### simdjson — The Performance Choice

```cpp

#include <simdjson.h>
#include <iostream>
#include <string>

void parse_with_simdjson() {
    simdjson::ondemand::parser parser;

    auto json_str = R"([
        {"name": "Alice", "score": 95.5},
        {"name": "Bob", "score": 87.3}
    ])"_padded;  // simdjson requires padded input

    auto doc = parser.iterate(json_str);
    for (auto item : doc.get_array()) {
        std::string_view name = item["name"].get_string();
        double score = item["score"].get_double();
        std::cout << name << ": " << score << "\n";
    }
    // Output:
    //   Alice: 95.5
    //   Bob: 87.3
}

// On-demand parsing: fields are parsed only when accessed
// ~10x faster than nlohmann for large documents

```

### glaze — Compile-Time Reflection (C++20)

```cpp

#include <glaze/glaze.hpp>
#include <iostream>

struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool debug = false;
};

// glaze uses compile-time reflection — no macros needed (C++20)
template<>
struct glz::meta<Config> {
    using T = Config;
    static constexpr auto value = object(
        "host", &T::host,
        "port", &T::port,
        "debug", &T::debug
    );
};

void use_glaze() {
    // Serialize
    Config cfg{"example.com", 443, true};
    std::string json = glz::write_json(cfg);
    std::cout << json << "\n";
    // {"host":"example.com","port":443,"debug":true}

    // Deserialize
    Config parsed;
    glz::read_json(parsed, json);
    std::cout << parsed.host << ":" << parsed.port << "\n";
}

```

---

## Self-Assessment

### Q1: When to choose each library

- **nlohmann/json**: Rapid prototyping, small data, schema-less exploration, excellent API
- **simdjson**: Parsing large JSON files (logs, API responses), read-only scenarios, maximum throughput
- **glaze**: Struct-based serialization, high performance with type safety, C++20 projects

### Q2: Show error handling with nlohmann/json

```cpp

#include <nlohmann/json.hpp>
#include <iostream>

void safe_parse() {
    try {
        auto j = nlohmann::json::parse("{invalid json}");
    } catch (const nlohmann::json::parse_error& e) {
        std::cerr << "Parse error: " << e.what() << "\n";
    }

    // Safe field access with .value() and defaults
    auto j = nlohmann::json::parse(R"({"name": "Alice"})");

    std::string name = j.value("name", "unknown");
    int age = j.value("age", -1);  // Missing field → default

    // Check field existence
    if (j.contains("email")) {
        std::cout << "Has email\n";
    }

    // Type-safe access with .get<T>()
    try {
        int wrong = j["name"].get<int>(); // Throws type_error
    } catch (const nlohmann::json::type_error& e) {
        std::cerr << "Type error: " << e.what() << "\n";
    }
}

```

### Q3: Parse a 10MB JSON file efficiently with simdjson on-demand

```cpp

#include <simdjson.h>

// On-demand parsing: only materializes values you access.
// For a 10MB file with 100K records, if you only need 1 field,
// ~99% of the data is never allocated.

void extract_field(const char* filename) {
    simdjson::ondemand::parser parser;
    auto json = simdjson::padded_string::load(filename);

    auto doc = parser.iterate(json);
    double total = 0;
    int count = 0;

    for (auto record : doc.get_array()) {
        // Only "score" is materialized — other fields are skipped
        double score = record["score"].get_double();
        total += score;
        ++count;
    }
    // Processes ~4 GB/s on modern hardware
}

```

---

## Notes

- nlohmann/json is the most widely used C++ JSON library — excellent for most use cases.
- simdjson exploits SIMD instructions (AVX2/NEON) for parsing — truly zero-allocation on-demand mode.
- glaze combines compile-time type safety with performance close to simdjson.
- For **writing** JSON, all three are roughly comparable; the gap is in parsing speed.
- Always validate and sanitize JSON from untrusted sources.
