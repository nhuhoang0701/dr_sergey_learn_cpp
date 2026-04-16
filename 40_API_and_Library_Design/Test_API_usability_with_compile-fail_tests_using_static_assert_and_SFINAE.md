# Test API usability with compile-fail tests using static_assert and SFINAE

**Category:** API & Library Design  
**Standard:** C++17/20  
**Reference:** <https://cmake.org/cmake/help/latest/prop_test/WILL_FAIL.html>  

---

## Topic Overview

Good APIs prevent misuse at compile time. **Compile-fail tests** verify that incorrect usage produces a compiler error — not silent bugs.

### static_assert Tests

```cpp

#include <type_traits>
#include <memory>

class NonCopyable {
public:
    NonCopyable() = default;
    NonCopyable(const NonCopyable&) = delete;
    NonCopyable& operator=(const NonCopyable&) = delete;
    NonCopyable(NonCopyable&&) = default;
    NonCopyable& operator=(NonCopyable&&) = default;
};

// Compile-time verification:
static_assert(!std::is_copy_constructible_v<NonCopyable>,
              "NonCopyable must not be copyable");
static_assert(std::is_move_constructible_v<NonCopyable>,
              "NonCopyable must be movable");
static_assert(std::is_nothrow_move_constructible_v<NonCopyable>,
              "NonCopyable move must be noexcept");

```

### CMake Compile-Fail Tests

```cmake

# CMakeLists.txt — test that certain code FAILS to compile

# Write a small source that should not compile
file(WRITE ${CMAKE_BINARY_DIR}/test_no_copy.cpp "
#include <mylib/handle.hpp>
int main() {
    mylib::Handle h1;
    mylib::Handle h2 = h1;  // Should FAIL — Handle is non-copyable
}
")

add_executable(test_no_copy EXCLUDE_FROM_ALL ${CMAKE_BINARY_DIR}/test_no_copy.cpp)
target_link_libraries(test_no_copy mylib)

add_test(NAME handle_no_copy
         COMMAND ${CMAKE_COMMAND} --build ${CMAKE_BINARY_DIR} --target test_no_copy)
set_tests_properties(handle_no_copy PROPERTIES WILL_FAIL TRUE)
# Test PASSES if compilation FAILS

```

### SFINAE-Based Trait Detection

```cpp

#include <type_traits>
#include <string>

// Detect if a type has a serialize() method
template<typename T, typename = void>
struct has_serialize : std::false_type {};

template<typename T>
struct has_serialize<T, std::void_t<decltype(std::declval<const T&>().serialize())>>
    : std::true_type {};

struct Good {
    std::string serialize() const { return "{}"; }
};

struct Bad {
    int value;
};

static_assert(has_serialize<Good>::value, "Good must be serializable");
static_assert(!has_serialize<Bad>::value, "Bad must NOT be serializable");

// C++20 concepts make this cleaner:
template<typename T>
concept Serializable20 = requires(const T& t) {
    { t.serialize() } -> std::convertible_to<std::string>;
};

static_assert(Serializable20<Good>);
static_assert(!Serializable20<Bad>);

```

---

## Self-Assessment

### Q1: Why test that code FAILS to compile

A library that says "Handle is non-copyable" should verify this in CI. If someone accidentally adds a copy constructor, the compile-fail test catches it. Without compile-fail tests, API guarantees are only documented, not enforced.

### Q2: Show concept-based compile-time diagnostics

```cpp

template<typename T>
concept HasNoThrowMove = std::is_nothrow_move_constructible_v<T>;

template<HasNoThrowMove T>
void store(std::vector<T>& vec, T&& item) {
    vec.push_back(std::move(item));
}

// Calling store with a type without noexcept move:
// error: constraints not satisfied [HasNoThrowMove<BadType>]

```

### Q3: How to test deleted overloads

```cpp

static_assert(!std::is_constructible_v<Buffer, int>,
              "Buffer(int) should be deleted");
static_assert(!std::is_constructible_v<Buffer, double>,
              "Buffer(double) should be deleted");
static_assert(std::is_constructible_v<Buffer, size_t>,
              "Buffer(size_t) should work");

```

---

## Notes

- Compile-fail tests are a form of **negative testing** — verifying incorrect usage is rejected.
- Use `static_assert` for same-TU checks and CMake `WILL_FAIL` for separate-compilation checks.
- C++20 concepts provide much better error messages than SFINAE when constraints are not met.
- Run compile-fail tests in CI to prevent accidental API contract violations.
