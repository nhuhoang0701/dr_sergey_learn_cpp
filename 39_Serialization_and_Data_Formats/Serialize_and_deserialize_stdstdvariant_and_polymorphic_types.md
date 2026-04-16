# Serialize and deserialize std::variant and polymorphic types

**Category:** Serialization & Data Formats  
**Standard:** C++17/20  
**Reference:** <https://en.cppreference.com/w/cpp/utility/variant>  

---

## Topic Overview

Serializing `std::variant` and polymorphic class hierarchies requires encoding which type is active. `std::variant` uses a type index; polymorphic types typically use a string type discriminator.

### Serializing std::variant

```cpp

#include <variant>
#include <string>
#include <iostream>
#include <cstdint>
#include <vector>
#include <cstring>

using Value = std::variant<int, double, std::string, bool>;

// Binary serialization with type tag
void serialize_value(const Value& v, std::vector<uint8_t>& buf) {
    uint8_t type_tag = static_cast<uint8_t>(v.index());
    buf.push_back(type_tag);

    std::visit([&buf](const auto& val) {
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, std::string>) {
            uint32_t len = static_cast<uint32_t>(val.size());
            auto p = reinterpret_cast<const uint8_t*>(&len);
            buf.insert(buf.end(), p, p + 4);
            buf.insert(buf.end(), val.begin(), val.end());
        } else {
            auto p = reinterpret_cast<const uint8_t*>(&val);
            buf.insert(buf.end(), p, p + sizeof(T));
        }
    }, v);
}

Value deserialize_value(const uint8_t*& pos) {
    uint8_t tag = *pos++;
    switch (tag) {
        case 0: { int v; std::memcpy(&v, pos, 4); pos += 4; return v; }
        case 1: { double v; std::memcpy(&v, pos, 8); pos += 8; return v; }
        case 2: {
            uint32_t len;
            std::memcpy(&len, pos, 4); pos += 4;
            std::string s(reinterpret_cast<const char*>(pos), len);
            pos += len;
            return s;
        }
        case 3: { bool v = (*pos++ != 0); return v; }
        default: throw std::runtime_error("Unknown type tag");
    }
}

int main() {
    std::vector<Value> values = {42, 3.14, std::string("hello"), true};
    std::vector<uint8_t> buf;

    for (auto& v : values) serialize_value(v, buf);
    std::cout << "Serialized: " << buf.size() << " bytes\n";

    const uint8_t* pos = buf.data();
    for (size_t i = 0; i < values.size(); ++i) {
        auto v = deserialize_value(pos);
        std::visit([](const auto& val) {
            std::cout << "  " << val << "\n";
        }, v);
    }
}

```

### Serializing Polymorphic Types with a Type Registry

```cpp

#include <memory>
#include <string>
#include <unordered_map>
#include <functional>
#include <iostream>

struct Shape {
    virtual ~Shape() = default;
    virtual std::string type_name() const = 0;
    virtual void serialize(std::ostream& os) const = 0;
    virtual void print() const = 0;
};

struct Circle : Shape {
    double radius;
    Circle(double r = 0) : radius(r) {}
    std::string type_name() const override { return "Circle"; }
    void serialize(std::ostream& os) const override { os << radius; }
    void print() const override { std::cout << "Circle(r=" << radius << ")"; }
};

struct Rect : Shape {
    double w, h;
    Rect(double w = 0, double h = 0) : w(w), h(h) {}
    std::string type_name() const override { return "Rect"; }
    void serialize(std::ostream& os) const override { os << w << " " << h; }
    void print() const override { std::cout << "Rect(" << w << "x" << h << ")"; }
};

// Factory registry for deserialization
using Factory = std::function<std::unique_ptr<Shape>(std::istream&)>;
std::unordered_map<std::string, Factory>& registry() {
    static std::unordered_map<std::string, Factory> r;
    return r;
}

template<typename T>
bool register_type(const std::string& name, Factory f) {
    registry()[name] = std::move(f);
    return true;
}

// Register types
static auto _c = register_type<Circle>("Circle", [](std::istream& is) {
    double r; is >> r; return std::make_unique<Circle>(r);
});
static auto _r = register_type<Rect>("Rect", [](std::istream& is) {
    double w, h; is >> w >> h; return std::make_unique<Rect>(w, h);
});

// Serialize: write type name + data
void save(std::ostream& os, const Shape& s) {
    os << s.type_name() << " ";
    s.serialize(os);
    os << "\n";
}

// Deserialize: read type name, look up factory
std::unique_ptr<Shape> load(std::istream& is) {
    std::string type;
    if (!(is >> type)) return nullptr;
    auto it = registry().find(type);
    if (it == registry().end()) throw std::runtime_error("Unknown type: " + type);
    return it->second(is);
}

```

---

## Self-Assessment

### Q1: Serialize a variant to JSON with type discrimination

```cpp

#include <variant>
#include <string>
#include <iostream>

using Value = std::variant<int, double, std::string>;

std::string variant_to_json(const Value& v) {
    return std::visit([](const auto& val) -> std::string {
        using T = std::decay_t<decltype(val)>;
        if constexpr (std::is_same_v<T, std::string>)
            return "{\"type\":\"string\",\"value\":\"" + val + "\"}";
        else if constexpr (std::is_same_v<T, int>)
            return "{\"type\":\"int\",\"value\":" + std::to_string(val) + "}";
        else
            return "{\"type\":\"double\",\"value\":" + std::to_string(val) + "}";
    }, v);
}

```

### Q2: Explain why virtual functions alone are insufficient for deserialization

Virtual functions dispatch on an existing object's type. During deserialization, no object exists yet — you need to **create** an object of the correct type from a type tag. This requires a factory/registry pattern external to the class hierarchy.

### Q3: Handle forward compatibility when new types are added to a variant

When a new alternative is added to the variant, old deserializers encounter an unknown type tag. Use a fallback strategy: skip the unknown data (if length-prefixed) or return a default/error value.

---

## Notes

- `std::variant::index()` is stable only if you never reorder alternatives — use explicit type tags instead.
- For polymorphic types, a type registry (factory map) is the standard pattern.
- nlohmann/json can serialize variants automatically with `NLOHMANN_JSON_SERIALIZE_ENUM`.
- Consider `std::any` if the set of types is truly unbounded (but you lose type safety).
