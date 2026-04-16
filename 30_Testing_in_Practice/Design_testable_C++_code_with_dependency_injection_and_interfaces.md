# Design testable C++ code with dependency injection and interfaces

**Category:** Testing in Practice

---

## Topic Overview

Writing testable C++ code begins at **design time**, not after implementation. The core principle: separate **what** your code does from **how** it interacts with external systems. This means coding against abstract interfaces and injecting concrete implementations, allowing test doubles to replace real I/O, network, database, and hardware.

### Testability Smells vs Fixes

| Untestable Pattern | Testable Alternative |
| --- | --- |
| `new` inside business logic | Constructor-inject a factory or ready-made object |
| Singleton for shared state | Inject the shared resource |
| Calling `std::time()` directly | Inject `IClock` interface |
| Direct file I/O in logic | Inject `IFileSystem` interface |
| Hard-coded URLs/paths | Inject configuration object |
| `static` functions with side effects | Wrap in mockable interface |

### Layers of Testable Architecture

```cpp

┌────────────────────────────────────────┐
│            Application / main()        │  ← Composition root: wires deps
├────────────────────────────────────────┤
│         Business Logic Layer           │  ← Pure logic, no I/O
│   (depends on abstract interfaces)     │  ← Easy to unit test
├────────────────────────────────────────┤
│     Infrastructure Adapters            │  ← Real implementations
│  (SqlRepo, HttpClient, FileSystem)     │  ← Integration tested
└────────────────────────────────────────┘

```

---

## Self-Assessment

### Q1: Refactor untestable code to be dependency-injectable

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <gmock/gmock.h>
#include <string>
#include <chrono>
#include <memory>

using ::testing::Return;
using ::testing::_;

// === BEFORE: Untestable — hard-coded dependencies ===
/*
class ReportGenerator {
public:
    std::string generate() {
        // Hard-coded: can't test without real DB and clock
        auto now = std::chrono::system_clock::now();
        auto data = SqlDatabase("prod://db").query("SELECT * FROM sales");
        std::ofstream file("/reports/" + format_date(now) + ".csv");
        file << format_csv(data);
        return file_path;
    }
};
*/

// === AFTER: Testable — all dependencies injected ===

// Interface for time
class IClock {
public:
    virtual ~IClock() = default;
    virtual std::string now_formatted() const = 0;
};

// Interface for data source
class IDataSource {
public:
    virtual ~IDataSource() = default;
    virtual std::vector<std::string> fetch_sales() = 0;
};

// Interface for file output
class IFileWriter {
public:
    virtual ~IFileWriter() = default;
    virtual bool write(const std::string& path, const std::string& content) = 0;
};

// Business logic: depends only on interfaces
class ReportGenerator {
public:
    ReportGenerator(IClock& clock, IDataSource& data, IFileWriter& writer)
        : clock_(clock), data_(data), writer_(writer) {}

    std::string generate() {
        auto timestamp = clock_.now_formatted();
        auto sales = data_.fetch_sales();

        std::string csv;
        for (const auto& row : sales)
            csv += row + "\n";

        std::string path = "/reports/" + timestamp + ".csv";
        if (!writer_.write(path, csv))
            throw std::runtime_error("Failed to write report");

        return path;
    }

private:
    IClock& clock_;
    IDataSource& data_;
    IFileWriter& writer_;
};

// === Mocks ===
class MockClock : public IClock {
public:
    MOCK_METHOD(std::string, now_formatted, (), (const, override));
};

class MockDataSource : public IDataSource {
public:
    MOCK_METHOD(std::vector<std::string>, fetch_sales, (), (override));
};

class MockFileWriter : public IFileWriter {
public:
    MOCK_METHOD(bool, write, (const std::string&, const std::string&), (override));
};

// === Tests: fast, deterministic, no I/O ===

TEST(ReportGeneratorTest, GeneratesCorrectPath) {
    MockClock clock;
    MockDataSource data;
    MockFileWriter writer;

    EXPECT_CALL(clock, now_formatted()).WillOnce(Return("2024-01-15"));
    EXPECT_CALL(data, fetch_sales()).WillOnce(Return(
        std::vector<std::string>{"widget,100", "gadget,200"}));
    EXPECT_CALL(writer, write("/reports/2024-01-15.csv", _))
        .WillOnce(Return(true));

    ReportGenerator gen(clock, data, writer);
    EXPECT_EQ(gen.generate(), "/reports/2024-01-15.csv");
}

TEST(ReportGeneratorTest, ThrowsOnWriteFailure) {
    MockClock clock;
    MockDataSource data;
    MockFileWriter writer;

    EXPECT_CALL(clock, now_formatted()).WillOnce(Return("2024-01-15"));
    EXPECT_CALL(data, fetch_sales()).WillOnce(Return(std::vector<std::string>{}));
    EXPECT_CALL(writer, write(_, _)).WillOnce(Return(false));

    ReportGenerator gen(clock, data, writer);
    EXPECT_THROW(gen.generate(), std::runtime_error);
}

```

### Q2: Compare testing with virtual interfaces vs template-based injection

**Answer:**

```cpp

// === Template-based DI: zero-cost, compile-time polymorphism ===

template<typename Clock, typename DataSource, typename FileWriter>
class ReportGeneratorT {
public:
    ReportGeneratorT(Clock& clock, DataSource& data, FileWriter& writer)
        : clock_(clock), data_(data), writer_(writer) {}

    std::string generate() {
        auto timestamp = clock_.now_formatted();
        auto sales = data_.fetch_sales();
        std::string csv;
        for (const auto& row : sales) csv += row + "\n";
        std::string path = "/reports/" + timestamp + ".csv";
        writer_.write(path, csv);
        return path;
    }

private:
    Clock& clock_;
    DataSource& data_;
    FileWriter& writer_;
};

// Test double: no vtable overhead
struct FakeClock {
    std::string now_formatted() const { return "2024-01-15"; }
};

struct FakeDataSource {
    std::vector<std::string> fetch_sales() { return {"a,1", "b,2"}; }
};

struct FakeFileWriter {
    std::string last_path;
    std::string last_content;
    bool write(const std::string& path, const std::string& content) {
        last_path = path;
        last_content = content;
        return true;
    }
};

TEST(TemplateDI, WritesCorrectContent) {
    FakeClock clock;
    FakeDataSource data;
    FakeFileWriter writer;

    ReportGeneratorT gen(clock, data, writer);
    gen.generate();

    EXPECT_EQ(writer.last_path, "/reports/2024-01-15.csv");
    EXPECT_EQ(writer.last_content, "a,1\nb,2\n");
}

```

| Approach | Overhead | Mockability | ABI Stable | Readability |
| --- | --- | --- | --- | --- |
| Virtual interface | vtable (1-3ns/call) | Google Mock | Yes | Familiar OOP |
| Template injection | Zero (inlined) | Manual fakes | No (header-only) | More verbose |
| `std::function` | Possible alloc | Lambda/functor | Yes | Best for single ops |

### Q3: Design a complete composition root with production and test wiring

**Answer:**

```cpp

#include <memory>
#include <iostream>

// === Production implementations ===
class SystemClock : public IClock {
public:
    std::string now_formatted() const override {
        // Real implementation using chrono
        return "2024-06-15";  // Simplified
    }
};

class SqlDataSource : public IDataSource {
public:
    explicit SqlDataSource(const std::string& connStr) : conn_(connStr) {}
    std::vector<std::string> fetch_sales() override {
        // Real SQL query
        return {"real,data"};
    }
private:
    std::string conn_;
};

class DiskFileWriter : public IFileWriter {
public:
    bool write(const std::string& path, const std::string& content) override {
        // Real file write
        std::cout << "Writing to " << path << "\n";
        return true;
    }
};

// === Composition Root: wires everything together ===

// Production composition (in main.cpp)
int main() {
    // Create infrastructure
    SystemClock clock;
    SqlDataSource data("prod://salesdb");
    DiskFileWriter writer;

    // Wire into business logic
    ReportGenerator generator(clock, data, writer);

    // Run
    auto path = generator.generate();
    std::cout << "Report generated: " << path << "\n";
}

// Test composition (in test file) — already shown above:
// MockClock, MockDataSource, MockFileWriter → ReportGenerator
// Fast, deterministic, no external dependencies

```

---

## Notes

- The **Composition Root** is the single place where all dependencies are wired — typically `main()`
- Never `#include` production implementations in test files — only interfaces
- Store injected dependencies as **references** (caller owns) or **`unique_ptr`** (class owns)
- For legacy code: start by wrapping the hardest-to-test dependency first (usually I/O)
- The [**Dependency Inversion Principle**](../29_OOP_Design/Apply_the_Dependency_Inversion_Principle_with_abstract_interfaces_and_dependency.md) is the theoretical foundation for DI
- Cost of virtual dispatch is negligible outside hot loops — don't prematurely optimize away interfaces
- Consider C++20 concepts as an alternative to interfaces for zero-cost DI with better error messages
