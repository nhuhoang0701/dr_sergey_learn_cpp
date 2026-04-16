# Write Concepts-constrained library interfaces with good diagnostics

**Category:** API & Library Design  
**Standard:** C++20  
**Reference:** <https://en.cppreference.com/w/cpp/language/constraints>  

---

## Topic Overview

Concepts replace SFINAE for constraining template parameters. They produce dramatically better error messages and make API requirements explicit in the signature.

### Before and After

```cpp

#include <concepts>
#include <type_traits>
#include <iostream>
#include <vector>
#include <string>

// BEFORE (SFINAE): cryptic "no matching function" errors
template<typename T,
         typename = std::enable_if_t<std::is_arithmetic_v<T>>>
T old_add(T a, T b) { return a + b; }

// AFTER (Concepts): clear "constraint not satisfied" message
template<std::integral T>
T add(T a, T b) { return a + b; }

// Custom concept with good diagnostics
template<typename T>
concept Serializable = requires(const T& t, std::ostream& os) {
    { t.serialize(os) } -> std::same_as<void>;
    { T::type_name() } -> std::convertible_to<std::string>;
};

template<Serializable T>
void save(const T& obj, std::ostream& os) {
    os << T::type_name() << ":";
    obj.serialize(os);
}

// Error message when constraint fails:
// "note: constraints not satisfied"
// "note: the required expression 't.serialize(os)' is invalid"
// vs SFINAE: "no matching function for call to 'save'"

```

### Layered Concepts for Clear Diagnostics

```cpp

#include <concepts>
#include <ranges>
#include <iterator>

// Build concepts incrementally — each layer adds one requirement
template<typename T>
concept HasSize = requires(const T& t) {
    { t.size() } -> std::convertible_to<std::size_t>;
};

template<typename T>
concept HasIterators = requires(T& t) {
    { t.begin() } -> std::input_or_output_iterator;
    { t.end() } -> std::sentinel_for<decltype(std::declval<T&>().begin())>;
};

template<typename T>
concept HasPushBack = requires(T& t, typename T::value_type v) {
    t.push_back(std::move(v));
};

template<typename T>
concept Collection = HasSize<T> && HasIterators<T>;

template<typename T>
concept GrowableCollection = Collection<T> && HasPushBack<T>;

// If Collection fails, the error says WHICH sub-concept failed:
// "note: 'HasSize<T>' was not satisfied"
// Much better than a monolithic concept with 5 requirements!

```

---

## Self-Assessment

### Q1: Write a concept for "JSON-serializable" types

```cpp

template<typename T>
concept JsonSerializable = requires(const T& t) {
    { to_json(t) } -> std::convertible_to<std::string>;
} && requires(const std::string& s) {
    { from_json<T>(s) } -> std::same_as<T>;
};

```

### Q2: Show how concept subsumption gives better overload selection

```cpp

template<typename T>
concept Printable = requires(const T& t, std::ostream& os) { os << t; };

template<typename T>
concept FormattedPrintable = Printable<T> && requires(const T& t) {
    { t.format() } -> std::convertible_to<std::string>;
};

// FormattedPrintable subsumes Printable → more specific overload wins
template<Printable T>
void print(const T& val) { std::cout << val; }

template<FormattedPrintable T>
void print(const T& val) { std::cout << val.format(); } // Preferred when both match

```

### Q3: Constrain return types with concepts

```cpp

template<typename Factory>
concept WidgetFactory = requires(Factory& f, int config) {
    { f.create(config) } -> std::derived_from<Widget>;
    { f.validate(config) } -> std::same_as<bool>;
};

```

---

## Notes

- Prefer layered, named concepts over deeply nested `requires` expressions.
- Concept names should read like adjectives: `Sortable`, `Serializable`, `Hashable`.
- Always test concept satisfaction with `static_assert(MyConcept<MyType>)`.
- Concepts are zero-cost at runtime — they only participate in overload resolution.
