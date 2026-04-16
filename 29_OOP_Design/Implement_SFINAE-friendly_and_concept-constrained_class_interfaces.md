# Implement SFINAE-friendly and concept-constrained class interfaces

**Category:** OOP Design

---

## Topic Overview

| Technique | Standard | Error Messages | Readability |
| --- | --- | --- | --- |
| SFINAE (`enable_if`) | C++11 | Terrible | Poor |
| `if constexpr` | C++17 | Better | Good |
| **Concepts** | C++20 | **Excellent** | **Great** |

### Evolution

```cpp

// C++11 SFINAE
template<typename T, typename = std::enable_if_t<std::is_integral_v<T>>>
void process(T val);

// C++20 Concepts
template<std::integral T>
void process(T val);  // Same constraint, 10x more readable

```

---

## Self-Assessment

### Q1: Show SFINAE vs concepts for constraining class methods

**Answer:**

```cpp

#include <type_traits>
#include <concepts>
#include <string>
#include <iostream>

// ===== C++11/14 SFINAE approach =====
template<typename T>
class ContainerOld {
    T data_;
public:
    explicit ContainerOld(T val) : data_(std::move(val)) {}

    // Only available for arithmetic types
    template<typename U = T>
    std::enable_if_t<std::is_arithmetic_v<U>, U>
    doubled() const { return data_ * 2; }

    // Only available for string-like types
    template<typename U = T>
    std::enable_if_t<std::is_convertible_v<U, std::string>, size_t>
    length() const { return std::string(data_).size(); }
};

// ===== C++20 Concepts approach =====
template<typename T>
class Container {
    T data_;
public:
    explicit Container(T val) : data_(std::move(val)) {}

    // Clean, readable constraints
    double doubled() const requires std::integral<T> || std::floating_point<T> {
        return data_ * 2;
    }

    size_t length() const requires std::convertible_to<T, std::string> {
        return std::string(data_).size();
    }

    // Unconstrained: available for all T
    const T& get() const { return data_; }
};

int main() {
    Container<int> ci(42);
    std::cout << ci.doubled() << "\n";  // 84
    // ci.length();  // COMPILE ERROR: int not convertible to string

    Container<std::string> cs("hello");
    std::cout << cs.length() << "\n";  // 5
    // cs.doubled();  // COMPILE ERROR: string not arithmetic
    return 0;
}

```

### Q2: Design a concept-constrained class with multiple constraint levels

**Answer:**

```cpp

#include <concepts>
#include <iostream>
#include <vector>
#include <string>

// Custom concepts
template<typename T>
concept Printable = requires(std::ostream& os, const T& t) {
    { os << t } -> std::same_as<std::ostream&>;
};

template<typename T>
concept Hashable = requires(T t) {
    { std::hash<T>{}(t) } -> std::convertible_to<size_t>;
};

template<typename T>
concept Summable = requires(T a, T b) {
    { a + b } -> std::same_as<T>;
};

// Class with graduated constraints
template<typename T>
class DataSeries {
    std::vector<T> data_;
public:
    void add(T val) { data_.push_back(std::move(val)); }
    size_t size() const { return data_.size(); }

    // Only for summable types
    T sum() const requires Summable<T> {
        T result{};
        for (const auto& v : data_) result = result + v;
        return result;
    }

    // Only for ordered types
    T max() const requires std::totally_ordered<T> {
        T result = data_.front();
        for (const auto& v : data_)
            if (v > result) result = v;
        return result;
    }

    // Only for printable types
    void print() const requires Printable<T> {
        for (const auto& v : data_) std::cout << v << " ";
        std::cout << "\n";
    }

    // Constrained friend
    friend std::ostream& operator<<(std::ostream& os, const DataSeries& ds)
        requires Printable<T> {
        os << "[";
        for (size_t i = 0; i < ds.data_.size(); ++i) {
            if (i > 0) os << ", ";
            os << ds.data_[i];
        }
        return os << "]";
    }
};

int main() {
    DataSeries<int> ints;
    ints.add(3); ints.add(1); ints.add(4);
    std::cout << "Sum: " << ints.sum() << "\n";  // 8
    std::cout << "Max: " << ints.max() << "\n";  // 4
    ints.print();  // 3 1 4

    DataSeries<std::string> strs;
    strs.add("hello"); strs.add("world");
    std::cout << "Sum: " << strs.sum() << "\n";  // helloworld
    strs.print();
    return 0;
}

```

### Q3: Show SFINAE-friendly trait for detecting capabilities

**Answer:**

```cpp

#include <type_traits>
#include <iostream>

// C++17 detection idiom
namespace detail {
    template<typename, typename = void>
    struct has_serialize : std::false_type {};

    template<typename T>
    struct has_serialize<T, std::void_t<
        decltype(std::declval<T>().serialize())
    >> : std::true_type {};

    template<typename, typename = void>
    struct has_size : std::false_type {};

    template<typename T>
    struct has_size<T, std::void_t<
        decltype(std::declval<T>().size())
    >> : std::true_type {};
}

template<typename T>
inline constexpr bool has_serialize_v = detail::has_serialize<T>::value;

template<typename T>
inline constexpr bool has_size_v = detail::has_size<T>::value;

// SFINAE-friendly process function
template<typename T>
void process(const T& obj) {
    if constexpr (has_serialize_v<T>) {
        std::cout << "Serialized: " << obj.serialize() << "\n";
    }
    if constexpr (has_size_v<T>) {
        std::cout << "Size: " << obj.size() << "\n";
    }
}

// C++20: same with concepts (much cleaner)
template<typename T>
concept HasSerialize = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

struct Widget {
    std::string serialize() const { return "widget-data"; }
};

struct Collection {
    size_t size() const { return 42; }
};

int main() {
    process(Widget{});      // Serialized: widget-data
    process(Collection{});  // Size: 42
    return 0;
}

```

---

## Notes

- **C++20 concepts are the default choice** for constraining templates — SFINAE only for legacy code
- `requires` clause on methods conditionally enables them based on template parameter
- Concepts give clear compiler errors: "constraint not satisfied" vs pages of SFINAE noise
- The detection idiom (`void_t` + specialization) is the C++17 SFINAE pattern for detecting members
- Use `if constexpr` for branching on type traits within a function body
- Subsumption: more constrained overloads are preferred over less constrained ones
