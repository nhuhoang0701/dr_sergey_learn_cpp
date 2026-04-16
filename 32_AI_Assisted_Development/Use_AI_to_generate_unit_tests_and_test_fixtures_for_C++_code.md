# Use AI to generate unit tests and test fixtures for C++ code

**Category:** AI-Assisted C++ Development

---

## Topic Overview

AI excels at generating **comprehensive test suites** from C++ code. Given a class or function, it can identify test scenarios including edge cases, error paths, and boundary conditions that developers might miss. The key is providing the right prompts: specify the testing framework, what to test, and the level of coverage expected.

### AI Test Generation Effectiveness

| Test Type | AI Quality | Human Review Needed |
| --- | --- | --- |
| Happy-path unit tests | Excellent | Low |
| Edge case identification | Good | Medium (may miss domain-specific) |
| Error path testing | Good | Medium |
| Concurrency tests | Weak | High |
| Performance/benchmark tests | Medium | Medium |
| Integration tests | Medium | High (needs system context) |
| Fuzz test harnesses | Good | Low |

---

## Self-Assessment

### Q1: Generate comprehensive unit tests from a class

**Answer:**

```cpp

=== PROMPT ===

"Generate GoogleTest unit tests for this C++ class.
Cover:

- Normal operation
- Edge cases (empty, single element, max capacity)
- Error conditions
- Move semantics
- Exception safety

Use TEST_F with a fixture for setup/teardown.
Each test should be independent and self-documenting.

```

```cpp

// === The class to test ===
template<typename T>
class RingBuffer {
public:
    explicit RingBuffer(size_t capacity);
    bool push(const T& item);
    bool push(T&& item);
    std::optional<T> pop();
    std::optional<T> peek() const;
    size_t size() const;
    size_t capacity() const;
    bool empty() const;
    bool full() const;
    void clear();
};

// === AI-generated test suite ===
class RingBufferTest : public ::testing::Test {
protected:
    void SetUp() override {
        buf_ = std::make_unique<RingBuffer<int>>(5);
    }
    std::unique_ptr<RingBuffer<int>> buf_;
};

// --- Construction ---
TEST_F(RingBufferTest, ConstructedEmpty) {
    EXPECT_TRUE(buf_->empty());
    EXPECT_FALSE(buf_->full());
    EXPECT_EQ(buf_->size(), 0);
    EXPECT_EQ(buf_->capacity(), 5);
}

// --- Push/Pop ---
TEST_F(RingBufferTest, PushPopSingle) {
    EXPECT_TRUE(buf_->push(42));
    EXPECT_EQ(buf_->size(), 1);
    auto val = buf_->pop();
    ASSERT_TRUE(val.has_value());
    EXPECT_EQ(*val, 42);
    EXPECT_TRUE(buf_->empty());
}

TEST_F(RingBufferTest, PushPopFIFO) {
    buf_->push(1); buf_->push(2); buf_->push(3);
    EXPECT_EQ(buf_->pop(), 1);
    EXPECT_EQ(buf_->pop(), 2);
    EXPECT_EQ(buf_->pop(), 3);
}

TEST_F(RingBufferTest, PushToFull) {
    for (int i = 0; i < 5; ++i)
        EXPECT_TRUE(buf_->push(i));
    EXPECT_TRUE(buf_->full());
    EXPECT_FALSE(buf_->push(99));  // Should fail
}

// --- Edge cases ---
TEST_F(RingBufferTest, PopFromEmpty) {
    auto val = buf_->pop();
    EXPECT_FALSE(val.has_value());
}

TEST_F(RingBufferTest, PeekDoesNotRemove) {
    buf_->push(42);
    auto peek1 = buf_->peek();
    auto peek2 = buf_->peek();
    EXPECT_EQ(peek1, peek2);
    EXPECT_EQ(buf_->size(), 1);
}

TEST_F(RingBufferTest, WrapAround) {
    // Fill then drain then fill to trigger wrap-around
    for (int i = 0; i < 5; ++i) buf_->push(i);
    for (int i = 0; i < 3; ++i) buf_->pop();
    for (int i = 10; i < 13; ++i) EXPECT_TRUE(buf_->push(i));

    EXPECT_EQ(buf_->pop(), 3);
    EXPECT_EQ(buf_->pop(), 4);
    EXPECT_EQ(buf_->pop(), 10);
    EXPECT_EQ(buf_->pop(), 11);
    EXPECT_EQ(buf_->pop(), 12);
}

TEST_F(RingBufferTest, ClearResetsState) {
    buf_->push(1); buf_->push(2);
    buf_->clear();
    EXPECT_TRUE(buf_->empty());
    EXPECT_EQ(buf_->size(), 0);
    EXPECT_TRUE(buf_->push(99));  // Can push after clear
}

// --- Move semantics ---
TEST_F(RingBufferTest, PushMoveOnly) {
    RingBuffer<std::unique_ptr<int>> mbuf(3);
    mbuf.push(std::make_unique<int>(42));
    auto val = mbuf.pop();
    ASSERT_NE(val, std::nullopt);
    EXPECT_EQ(**val, 42);
}

```

### Q2: Generate test fixtures with mock objects

**Answer:**

```cpp

=== PROMPT ===

"Generate GoogleTest + GoogleMock tests for OrderService.
Mock out the database and email dependencies.
Test each method with success and failure scenarios."

```

```cpp

// === Interfaces to mock ===
class IOrderRepository {
public:
    virtual ~IOrderRepository() = default;
    virtual std::optional<Order> find(const std::string& id) = 0;
    virtual bool save(const Order& order) = 0;
    virtual bool remove(const std::string& id) = 0;
};

class IEmailService {
public:
    virtual ~IEmailService() = default;
    virtual bool send(const std::string& to, const std::string& subject,
                      const std::string& body) = 0;
};

// === AI-generated mocks ===
class MockOrderRepo : public IOrderRepository {
public:
    MOCK_METHOD(std::optional<Order>, find, (const std::string&), (override));
    MOCK_METHOD(bool, save, (const Order&), (override));
    MOCK_METHOD(bool, remove, (const std::string&), (override));
};

class MockEmailService : public IEmailService {
public:
    MOCK_METHOD(bool, send, (const std::string&, const std::string&,
                              const std::string&), (override));
};

// === AI-generated fixture ===
class OrderServiceTest : public ::testing::Test {
protected:
    void SetUp() override {
        repo_ = std::make_shared<MockOrderRepo>();
        email_ = std::make_shared<MockEmailService>();
        service_ = std::make_unique<OrderService>(repo_, email_);
    }

    std::shared_ptr<MockOrderRepo> repo_;
    std::shared_ptr<MockEmailService> email_;
    std::unique_ptr<OrderService> service_;

    Order sample_order() {
        return Order{"ORD-001", "customer@test.com", 99.99};
    }
};

// === Tests ===
TEST_F(OrderServiceTest, CreateOrder_Success) {
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(true));
    EXPECT_CALL(*email_, send("customer@test.com", ::testing::_, ::testing::_))
        .WillOnce(::testing::Return(true));

    auto result = service_->create_order("customer@test.com", 99.99);
    ASSERT_TRUE(result.has_value());
    EXPECT_FALSE(result->id.empty());
}

TEST_F(OrderServiceTest, CreateOrder_DbFailure) {
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(false));
    EXPECT_CALL(*email_, send(::testing::_, ::testing::_, ::testing::_))
        .Times(0);  // Email should NOT be sent on save failure

    auto result = service_->create_order("customer@test.com", 99.99);
    EXPECT_FALSE(result.has_value());
}

TEST_F(OrderServiceTest, CancelOrder_NotFound) {
    EXPECT_CALL(*repo_, find("ORD-999"))
        .WillOnce(::testing::Return(std::nullopt));

    auto result = service_->cancel_order("ORD-999");
    EXPECT_FALSE(result);
}

TEST_F(OrderServiceTest, CancelOrder_EmailFailure_StillCancels) {
    EXPECT_CALL(*repo_, find("ORD-001"))
        .WillOnce(::testing::Return(sample_order()));
    EXPECT_CALL(*repo_, save(::testing::_))
        .WillOnce(::testing::Return(true));
    EXPECT_CALL(*email_, send(::testing::_, ::testing::_, ::testing::_))
        .WillOnce(::testing::Return(false));  // Email fails

    auto result = service_->cancel_order("ORD-001");
    EXPECT_TRUE(result);  // Order still cancelled despite email failure
}

```

### Q3: Generate edge case and boundary tests

**Answer:**

```cpp

=== PROMPT ===

"For this function, generate tests for ALL edge cases:

- Empty input
- Single element
- Boundary values (INT_MIN, INT_MAX, 0)
- Duplicate values
- Already sorted / reverse sorted
- Large input (performance sanity check)

```

```cpp

// Function to test:
int find_median(std::vector<int>& data);

// AI-generated edge case tests:
TEST(FindMedianTest, EmptyVector_Throws) {
    std::vector<int> empty;
    EXPECT_THROW(find_median(empty), std::invalid_argument);
}

TEST(FindMedianTest, SingleElement) {
    std::vector<int> v{42};
    EXPECT_EQ(find_median(v), 42);
}

TEST(FindMedianTest, TwoElements) {
    std::vector<int> v{1, 3};
    EXPECT_EQ(find_median(v), 2);  // Average for even count
}

TEST(FindMedianTest, OddCount) {
    std::vector<int> v{3, 1, 2};
    EXPECT_EQ(find_median(v), 2);
}

TEST(FindMedianTest, EvenCount) {
    std::vector<int> v{4, 1, 3, 2};
    EXPECT_EQ(find_median(v), 2);  // (2+3)/2 = 2 (integer division)
}

TEST(FindMedianTest, AllDuplicates) {
    std::vector<int> v{5, 5, 5, 5, 5};
    EXPECT_EQ(find_median(v), 5);
}

TEST(FindMedianTest, AlreadySorted) {
    std::vector<int> v{1, 2, 3, 4, 5};
    EXPECT_EQ(find_median(v), 3);
}

TEST(FindMedianTest, ReverseSorted) {
    std::vector<int> v{5, 4, 3, 2, 1};
    EXPECT_EQ(find_median(v), 3);
}

TEST(FindMedianTest, NegativeNumbers) {
    std::vector<int> v{-5, -1, -3};
    EXPECT_EQ(find_median(v), -3);
}

TEST(FindMedianTest, MixedSignedNumbers) {
    std::vector<int> v{-10, 0, 10};
    EXPECT_EQ(find_median(v), 0);
}

TEST(FindMedianTest, IntBoundaries) {
    std::vector<int> v{INT_MIN, 0, INT_MAX};
    EXPECT_EQ(find_median(v), 0);
}

TEST(FindMedianTest, LargeInput_Performance) {
    std::vector<int> v(1'000'000);
    std::iota(v.begin(), v.end(), 0);
    std::shuffle(v.begin(), v.end(), std::mt19937{42});

    auto start = std::chrono::steady_clock::now();
    auto result = find_median(v);
    auto ms = std::chrono::duration_cast<std::chrono::milliseconds>(
        std::chrono::steady_clock::now() - start).count();

    EXPECT_EQ(result, 499'999);
    EXPECT_LT(ms, 1000);  // Should complete in <1s
}

```

---

## Notes

- **Prompt with the class interface** (not just the name) for best results
- Ask AI to generate tests **before** implementation (TDD workflow)
- AI excels at **edge cases** — it systematically considers empty, single, boundary, overflow
- **Mock tests** need interface definitions — include them in the prompt
- Always **review AI-generated test assertions** — they may assert the wrong expected value
- Use **parameterized tests** (`TEST_P`) for data-driven testing of multiple inputs
- AI is weak at **concurrency tests** — it often generates tests that pass by luck
