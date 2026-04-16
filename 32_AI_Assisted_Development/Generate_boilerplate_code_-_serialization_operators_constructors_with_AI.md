# Generate boilerplate code - serialization, operators, constructors with AI

**Category:** AI-Assisted C++ Development

---

## Topic Overview

C++ requires significant **boilerplate code**: special member functions, comparison operators, serialization, stream operators, hash functions, and swap specializations. AI excels at generating this code because it follows mechanical patterns. The key is specifying exactly what you need and the coding style/conventions to follow.

### Boilerplate Types and AI Generation Quality

| Boilerplate Type | AI Quality | Risk to Watch |
| --- | --- | --- |
| Rule of Five (copy/move ctors/assign/dtor) | Excellent | Move semantics correctness |
| operator<< for debugging | Excellent | Format consistency |
| operator<=> (spaceship) | Good | Weak ordering vs strong ordering |
| JSON serialization (nlohmann) | Excellent | Missing fields, null handling |
| Protobuf converters | Good | Oneof handling |
| Hash function (std::hash) | Good | Hash quality for custom types |
| Builder pattern | Excellent | Template deduction issues |
| Visitor pattern boilerplate | Good | std::visit with variants |

---

## Self-Assessment

### Q1: Generate special member functions and operators

**Answer:**

```cpp

=== PROMPT ===

"Generate the Rule of Five, operator==, operator<=>,
operator<<, and std::hash specialization for this class.
Use C++20 features. Follow Google Style Guide."

```

```cpp

// Input class:
class Employee {
    std::string name_;
    int id_;
    std::vector<std::string> skills_;
    std::unique_ptr<Address> address_;  // Deep copy needed
public:
    Employee(std::string name, int id);
};

// === AI-generated boilerplate ===

class Employee {
    std::string name_;
    int id_;
    std::vector<std::string> skills_;
    std::unique_ptr<Address> address_;

public:
    // Constructor
    Employee(std::string name, int id)
        : name_(std::move(name)), id_(id) {}

    // Copy constructor (deep copy)
    Employee(const Employee& other)
        : name_(other.name_)
        , id_(other.id_)
        , skills_(other.skills_)
        , address_(other.address_
                   ? std::make_unique<Address>(*other.address_)
                   : nullptr) {}

    // Copy assignment (deep copy via copy-and-swap)
    Employee& operator=(const Employee& other) {
        if (this != &other) {
            Employee tmp(other);
            swap(*this, tmp);
        }
        return *this;
    }

    // Move constructor
    Employee(Employee&&) noexcept = default;

    // Move assignment
    Employee& operator=(Employee&&) noexcept = default;

    // Destructor
    ~Employee() = default;

    // Equality (C++20)
    bool operator==(const Employee& other) const {
        return id_ == other.id_
            && name_ == other.name_
            && skills_ == other.skills_
            && (address_ == other.address_
                || (address_ && other.address_
                    && *address_ == *other.address_));
    }

    // Ordering (by ID, then name)
    std::strong_ordering operator<=>(const Employee& other) const {
        if (auto cmp = id_ <=> other.id_; cmp != 0) return cmp;
        return name_ <=> other.name_;
    }

    // Stream output
    friend std::ostream& operator<<(std::ostream& os, const Employee& e) {
        os << "Employee{id=" << e.id_ << ", name=\"" << e.name_
           << "\", skills=[";
        for (size_t i = 0; i < e.skills_.size(); ++i) {
            if (i > 0) os << ", ";
            os << '"' << e.skills_[i] << '"';
        }
        os << "]}";
        return os;
    }

    // Swap (for copy-and-swap idiom)
    friend void swap(Employee& a, Employee& b) noexcept {
        using std::swap;
        swap(a.name_, b.name_);
        swap(a.id_, b.id_);
        swap(a.skills_, b.skills_);
        swap(a.address_, b.address_);
    }

    // Accessors for hash
    int id() const { return id_; }
    const std::string& name() const { return name_; }
};

// std::hash specialization
template<>
struct std::hash<Employee> {
    size_t operator()(const Employee& e) const noexcept {
        size_t h = std::hash<int>{}(e.id());
        h ^= std::hash<std::string>{}(e.name()) + 0x9e3779b9

             + (h << 6) + (h >> 2);

        return h;
    }
};

```

### Q2: Generate serialization code

**Answer:**

```cpp

=== PROMPT ===

"Generate nlohmann::json serialization (to_json/from_json)
for this class hierarchy. Handle:

- Optional fields (std::optional)
- Enum serialization as strings
- Variant fields
- Null/missing field handling with defaults"

```

```cpp

enum class Priority { Low, Medium, High, Critical };

struct Task {
    std::string title;
    Priority priority;
    std::optional<std::string> assignee;
    std::optional<std::chrono::system_clock::time_point> deadline;
    std::vector<std::string> tags;
    std::variant<int, std::string> reference;  // ticket # or URL
};

// === AI-generated serialization ===

// Enum string mapping
NLOHMANN_JSON_SERIALIZE_ENUM(Priority, {
    {Priority::Low, "low"},
    {Priority::Medium, "medium"},
    {Priority::High, "high"},
    {Priority::Critical, "critical"},
})

// Custom time_point serialization (ISO 8601)
inline void to_json(nlohmann::json& j,
                    const std::chrono::system_clock::time_point& tp) {
    auto t = std::chrono::system_clock::to_time_t(tp);
    std::ostringstream ss;
    ss << std::put_time(std::gmtime(&t), "%FT%TZ");
    j = ss.str();
}

inline void from_json(const nlohmann::json& j,
                      std::chrono::system_clock::time_point& tp) {
    std::tm tm = {};
    std::istringstream ss(j.get<std::string>());
    ss >> std::get_time(&tm, "%Y-%m-%dT%H:%M:%S");
    tp = std::chrono::system_clock::from_time_t(std::mktime(&tm));
}

// Task serialization
inline void to_json(nlohmann::json& j, const Task& t) {
    j = nlohmann::json{
        {"title", t.title},
        {"priority", t.priority},
        {"tags", t.tags}
    };
    if (t.assignee) j["assignee"] = *t.assignee;
    if (t.deadline) j["deadline"] = *t.deadline;

    // Variant serialization
    std::visit([&j](const auto& ref) {
        using T = std::decay_t<decltype(ref)>;
        if constexpr (std::is_same_v<T, int>)
            j["reference"] = {{"type", "ticket"}, {"value", ref}};
        else
            j["reference"] = {{"type", "url"}, {"value", ref}};
    }, t.reference);
}

inline void from_json(const nlohmann::json& j, Task& t) {
    j.at("title").get_to(t.title);
    j.at("priority").get_to(t.priority);
    t.tags = j.value("tags", std::vector<std::string>{});

    if (j.contains("assignee") && !j["assignee"].is_null())
        t.assignee = j["assignee"].get<std::string>();
    if (j.contains("deadline") && !j["deadline"].is_null())
        t.deadline = j["deadline"].get<
            std::chrono::system_clock::time_point>();

    // Variant deserialization
    if (j.contains("reference")) {
        auto type = j["reference"]["type"].get<std::string>();
        if (type == "ticket")
            t.reference = j["reference"]["value"].get<int>();
        else
            t.reference = j["reference"]["value"].get<std::string>();
    }
}

```

### Q3: Generate builder pattern and factory boilerplate

**Answer:**

```cpp

=== PROMPT ===

"Generate a fluent Builder pattern for this config class.
Use method chaining, validate on build(), return
std::expected<Config, std::string> from build()."

```

```cpp

// === AI-generated builder ===
struct ServerConfig {
    std::string host;
    uint16_t port;
    size_t max_connections;
    std::chrono::seconds timeout;
    bool tls_enabled;
    std::optional<std::string> cert_path;
    std::optional<std::string> key_path;
};

class ServerConfigBuilder {
    ServerConfig config_{};

public:
    ServerConfigBuilder& host(std::string h) {
        config_.host = std::move(h); return *this;
    }
    ServerConfigBuilder& port(uint16_t p) {
        config_.port = p; return *this;
    }
    ServerConfigBuilder& max_connections(size_t n) {
        config_.max_connections = n; return *this;
    }
    ServerConfigBuilder& timeout(std::chrono::seconds t) {
        config_.timeout = t; return *this;
    }
    ServerConfigBuilder& enable_tls(
            std::string cert, std::string key) {
        config_.tls_enabled = true;
        config_.cert_path = std::move(cert);
        config_.key_path = std::move(key);
        return *this;
    }

    std::expected<ServerConfig, std::string> build() const {
        if (config_.host.empty())
            return std::unexpected("Host is required");
        if (config_.port == 0)
            return std::unexpected("Port must be non-zero");
        if (config_.max_connections == 0)
            return std::unexpected("Max connections must be > 0");
        if (config_.tls_enabled) {
            if (!config_.cert_path || !config_.key_path)
                return std::unexpected(
                    "TLS requires both cert and key paths");
        }
        return config_;
    }
};

// Usage:
auto config = ServerConfigBuilder()
    .host("0.0.0.0")
    .port(8080)
    .max_connections(1000)
    .timeout(std::chrono::seconds{30})
    .enable_tls("/etc/ssl/cert.pem", "/etc/ssl/key.pem")
    .build();

if (!config) {
    std::cerr << "Config error: " << config.error() << '\n';
    return 1;
}

```

---

## Notes

- Always specify **which C++ standard** and **coding style** in the prompt
- For `operator<=>`, tell AI whether you need **strong, weak, or partial** ordering
- AI-generated `std::hash` may have poor distribution — review for hash quality
- For serialization, explicitly mention **how to handle missing/null fields**
- **Don't accept unique_ptr copy** in AI output — common AI mistake (unique_ptr is move-only)
- Ask AI to generate boilerplate for **existing code** by pasting the class definition
- For large classes, ask AI to generate boilerplate **incrementally** (operators first, then serialization)
