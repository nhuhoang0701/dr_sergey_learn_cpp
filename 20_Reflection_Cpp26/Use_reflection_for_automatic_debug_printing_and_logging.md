# Use reflection for automatic debug printing and logging

**Category:** Reflection (C++26)  
**Standard:** C++26  
**Reference:** <https://wg21.link/P2996>  

---

## Topic Overview

### The Problem

In C++, printing/logging structs for debugging requires manual boilerplate:

```cpp

struct Config {
    std::string host;
    int port;
    bool tls;
};

// You have to write this by hand for every struct:
std::ostream& operator<<(std::ostream& os, const Config& c) {
    return os << "Config{host=" << c.host
              << ", port=" << c.port
              << ", tls=" << c.tls << "}";
}

```

With reflection, this becomes automatic.

### Automatic `operator<<` with Reflection

```cpp

#include <meta>
#include <iostream>
#include <string>

template<typename T>
    requires std::is_aggregate_v<T>
std::ostream& auto_print(std::ostream& os, const T& obj) {
    os << std::meta::display_string_of(^^T) << "{";
    bool first = true;

    template for (constexpr auto member :
                  std::meta::nonstatic_data_members_of(^^T)) {
        if (!first) os << ", ";
        os << std::meta::identifier_of(member) << "="
           << obj.[:member:];
        first = false;
    }

    return os << "}";
}

struct Config {
    std::string host = "localhost";
    int port = 8080;
    bool tls = true;
};

int main() {
    Config c;
    auto_print(std::cout, c);
    // Output: Config{host=localhost, port=8080, tls=1}
}

```

### Generic Debug Logger

```cpp

#include <meta>
#include <format>
#include <source_location>
#include <iostream>

template<typename T>
std::string to_debug_string(const T& obj) {
    if constexpr (std::is_aggregate_v<T>) {
        std::string result = std::string(std::meta::display_string_of(^^T)) + "{";
        bool first = true;

        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            if (!first) result += ", ";
            result += std::meta::identifier_of(m);
            result += "=";
            result += std::format("{}", obj.[:m:]);
            first = false;
        }

        result += "}";
        return result;
    } else {
        return std::format("{}", obj);
    }
}

#define LOG_DEBUG(obj) \
    std::cerr << "[DEBUG " << std::source_location::current().file_name() \
              << ":" << std::source_location::current().line() << "] " \
              << to_debug_string(obj) << "\n"

```

### Recursive Printing for Nested Structs

```cpp

template<typename T>
std::string deep_debug(const T& obj, int indent = 0) {
    std::string pad(indent * 2, ' ');

    if constexpr (std::is_aggregate_v<T>) {
        std::string result = std::string(std::meta::display_string_of(^^T)) + " {\n";

        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            result += pad + "  " + std::string(std::meta::identifier_of(m)) + " = ";
            result += deep_debug(obj.[:m:], indent + 1);
            result += "\n";
        }

        result += pad + "}";
        return result;
    } else {
        return std::format("{}", obj);
    }
}

struct Address {
    std::string city;
    int zip;
};

struct Person {
    std::string name;
    int age;
    Address address;
};

int main() {
    Person p{"Alice", 30, {"NYC", 10001}};
    std::cout << deep_debug(p) << "\n";
    // Output:
    // Person {
    //   name = Alice
    //   age = 30
    //   address = Address {
    //     city = NYC
    //     zip = 10001
    //   }
    // }
}

```

### Structured Logging with Key-Value Pairs

```cpp

template<typename T>
auto to_log_map(const T& obj) {
    std::vector<std::pair<std::string, std::string>> entries;

    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        entries.emplace_back(
            std::string(std::meta::identifier_of(m)),
            std::format("{}", obj.[:m:])
        );
    }

    return entries;
}

// Integration with structured logging systems:
// auto kv = to_log_map(config);
// logger.info("Server starting", kv);

```

---

## Self-Assessment

### Q1: Write a generic `to_string` function using reflection

```cpp

template<typename T>
std::string auto_to_string(const T& obj) {
    if constexpr (std::is_aggregate_v<T>) {
        std::string result = "{";
        bool first = true;
        template for (constexpr auto m :
                      std::meta::nonstatic_data_members_of(^^T)) {
            if (!first) result += ", ";
            result += std::format("{}={}", std::meta::identifier_of(m), obj.[:m:]);
            first = false;
        }
        return result + "}";
    } else {
        return std::format("{}", obj);
    }
}

```

### Q2: How does `template for` differ from a regular `for` loop

`template for` is a **compile-time expansion** loop (sometimes called "expansion statements"). Each iteration is instantiated separately, allowing:

- Different types per iteration (each member may have a different type).
- Compile-time access to reflection values.
- No runtime overhead — the loop is fully unrolled at compile time.

A regular `for` loop iterates at runtime over a homogeneous container.

### Q3: Generate an `operator==` automatically using reflection

```cpp

template<typename T>
    requires std::is_aggregate_v<T>
bool auto_equals(const T& a, const T& b) {
    bool equal = true;
    template for (constexpr auto m :
                  std::meta::nonstatic_data_members_of(^^T)) {
        equal = equal && (a.[:m:] == b.[:m:]);
    }
    return equal;
}

```

---

## Notes

- Reflection-based debug printing eliminates hundreds of lines of boilerplate in real codebases.
- The `template for` / expansion statement syntax may evolve — check the latest P2996 revision.
- For production logging, consider generating `std::format` specializations instead of `operator<<`.
- Reflection-based printing integrates naturally with structured logging frameworks.
