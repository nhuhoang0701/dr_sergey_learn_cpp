# Test template-heavy code with explicit instantiation and type lists

**Category:** Testing in Practice

---

## Topic Overview

Testing template code is challenging because templates are only instantiated when used. Untested template specializations may contain bugs that only appear when instantiated with specific types. Strategies include **explicit instantiation** (force compilation), **type-parameterized tests** (test across many types), and **concept checks** (compile-time validation).

### Testing Challenges for Templates

| Challenge | Strategy |
| --- | --- |
| Template not instantiated = not compiled | Explicit instantiation in test TU |
| Works for `int` but fails for `std::string` | Type-parameterized tests across type list |
| SFINAE failure is silent | Static assertions + concept checks |
| Template error messages are unreadable | Test compilation failures with `static_assert` |
| Partial specialization bugs | Test each specialization with representative types |

---

## Self-Assessment

### Q1: Use type-parameterized tests to verify template code across multiple types

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <type_traits>
#include <limits>

// === Template code under test ===
template<typename T>
class RingBuffer {
public:
    explicit RingBuffer(size_t capacity)
        : data_(capacity), capacity_(capacity) {}

    void push(const T& value) {
        data_[write_pos_ % capacity_] = value;
        ++write_pos_;
        if (size_ < capacity_) ++size_;
    }

    T pop() {
        if (size_ == 0) throw std::underflow_error("Empty");
        T val = data_[read_pos_ % capacity_];
        ++read_pos_;
        --size_;
        return val;
    }

    size_t size() const { return size_; }
    bool empty() const { return size_ == 0; }
    bool full() const { return size_ == capacity_; }

private:
    std::vector<T> data_;
    size_t capacity_;
    size_t write_pos_ = 0;
    size_t read_pos_ = 0;
    size_t size_ = 0;
};

// === Type-parameterized test suite ===

template<typename T>
class RingBufferTest : public ::testing::Test {
protected:
    RingBuffer<T> buffer{5};  // Capacity 5

    // Helper: create a test value of type T
    static T make_value(int n) {
        if constexpr (std::is_arithmetic_v<T>)
            return static_cast<T>(n);
        else if constexpr (std::is_same_v<T, std::string>)
            return "item_" + std::to_string(n);
        else
            return T{};  // Default constructible
    }
};

// Test with representative types: integral, floating, string, complex
using TestTypes = ::testing::Types<
    int, unsigned, double, float, std::string, int64_t
>;

TYPED_TEST_SUITE(RingBufferTest, TestTypes);

TYPED_TEST(RingBufferTest, StartsEmpty) {
    EXPECT_TRUE(this->buffer.empty());
    EXPECT_EQ(this->buffer.size(), 0);
}

TYPED_TEST(RingBufferTest, PushIncreasesSize) {
    this->buffer.push(this->make_value(1));
    EXPECT_EQ(this->buffer.size(), 1);
    EXPECT_FALSE(this->buffer.empty());
}

TYPED_TEST(RingBufferTest, PushPopReturnsCorrectValue) {
    auto val = this->make_value(42);
    this->buffer.push(val);
    EXPECT_EQ(this->buffer.pop(), val);
}

TYPED_TEST(RingBufferTest, OverwritesOldestWhenFull) {
    for (int i = 0; i < 7; ++i)  // Push 7 into capacity-5
        this->buffer.push(this->make_value(i));

    EXPECT_TRUE(this->buffer.full());
    // Oldest (0, 1) overwritten; 2 is now oldest readable
    EXPECT_EQ(this->buffer.pop(), this->make_value(2));
}

TYPED_TEST(RingBufferTest, PopFromEmptyThrows) {
    EXPECT_THROW(this->buffer.pop(), std::underflow_error);
}

```

### Q2: Use explicit instantiation to force template compilation

**Answer:**

```cpp

// === Force instantiation of all template methods ===
// Even if tests don't call every method, explicit instantiation
// ensures the code compiles for those types.

// In a test file or dedicated instantiation file:
template class RingBuffer<int>;
template class RingBuffer<double>;
template class RingBuffer<std::string>;
template class RingBuffer<std::vector<int>>;  // Nested containers too

// This catches errors like:
// - Missing includes for specific type operations
// - Accidentally using int-only operations (e.g., %) on non-integral types
// - Missing move/copy constructors for stored types


// === Compile-time testing with static_assert ===

template<typename T>
concept RingBufferCompatible = requires {
    requires std::is_default_constructible_v<T>;
    requires std::is_copy_assignable_v<T>;
    requires std::is_move_constructible_v<T>;
};

// Verify concept at compile time:
static_assert(RingBufferCompatible<int>);
static_assert(RingBufferCompatible<std::string>);
// static_assert(RingBufferCompatible<std::unique_ptr<int>>);  // FAILS: not copyable


// === Test template specializations ===

template<typename T>
struct Serializer {
    static std::string serialize(const T& val) {
        return std::to_string(val);  // Generic: works for numeric types
    }
};

// Specialization for string
template<>
struct Serializer<std::string> {
    static std::string serialize(const std::string& val) {
        return '"' + val + '"';
    }
};

// Test BOTH the generic template AND the specialization:
TEST(SerializerTest, IntSerialization) {
    EXPECT_EQ(Serializer<int>::serialize(42), "42");
}

TEST(SerializerTest, DoubleSerialization) {
    EXPECT_THAT(Serializer<double>::serialize(3.14),
                ::testing::HasSubstr("3.14"));
}

TEST(SerializerTest, StringSpecialization) {
    EXPECT_EQ(Serializer<std::string>::serialize("hello"), "\"hello\"");
}

```

### Q3: Test SFINAE expressions and concept-constrained templates

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <type_traits>
#include <concepts>

// === Template with SFINAE: only enabled for specific types ===

template<typename T>
auto safe_divide(T a, T b)
    -> std::enable_if_t<std::is_arithmetic_v<T>, T>
{
    if constexpr (std::is_integral_v<T>) {
        if (b == 0) throw std::domain_error("Division by zero");
    }
    return a / b;
}

// Verify SFINAE works: this should NOT compile for non-arithmetic types
// Test at compile time:
static_assert(std::is_invocable_v<decltype(safe_divide<int>), int, int>);
static_assert(std::is_invocable_v<decltype(safe_divide<double>), double, double>);
// static_assert(!std::is_invocable_v<decltype(safe_divide<std::string>), std::string, std::string>);
// ^ Can't easily test this without more machinery

// Runtime tests for all enabled types:
TEST(SafeDivideTest, IntegerDivision) {
    EXPECT_EQ(safe_divide(10, 3), 3);  // Integer division
}

TEST(SafeDivideTest, FloatingDivision) {
    EXPECT_DOUBLE_EQ(safe_divide(10.0, 3.0), 10.0 / 3.0);
}

TEST(SafeDivideTest, IntegerDivisionByZeroThrows) {
    EXPECT_THROW(safe_divide(10, 0), std::domain_error);
}

TEST(SafeDivideTest, FloatingDivisionByZeroProducesInfinity) {
    // Floating-point division by zero is well-defined: produces infinity
    EXPECT_TRUE(std::isinf(safe_divide(10.0, 0.0)));
}


// === Concept-constrained template ===

template<typename T>
concept Printable = requires(T t, std::ostream& os) {
    { os << t } -> std::convertible_to<std::ostream&>;
};

template<Printable T>
std::string to_debug_string(const T& val) {
    std::ostringstream oss;
    oss << val;
    return oss.str();
}

// Compile-time concept tests:
static_assert(Printable<int>);
static_assert(Printable<std::string>);
static_assert(Printable<double>);
static_assert(!Printable<std::vector<int>>);  // No operator<< by default

TEST(ConceptTest, PrintableTypesWork) {
    EXPECT_EQ(to_debug_string(42), "42");
    EXPECT_EQ(to_debug_string(std::string("hello")), "hello");
}

```

---

## Notes

- **Explicit instantiation** is the simplest way to catch template compilation errors for untested types
- Use `TYPED_TEST_SUITE` for testing the same logic across a type list (gtest) or `GENERATE` (Catch2)
- `static_assert` tests concepts and type traits at compile time — zero runtime cost
- Test **each specialization** separately; partial specializations can have unique bugs
- For SFINAE, test both the "enabled" case (works) and "disabled" case (doesn't compile / concept fails)
- Template error messages improve dramatically with C++20 concepts — use them
- Consider a dedicated `instantiation_test.cpp` that explicitly instantiates all template combinations
