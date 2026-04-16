# Understand test isolation and avoid shared mutable state in test suites

**Category:** Testing & Verification  
**Item:** #770  
**Reference:** <https://github.com/google/googletest>  

---

## Topic Overview

**Test isolation** means each test runs independently — no test's outcome depends on another test running first. The primary enemy is **shared mutable state**: globals, static variables, singletons, files, and databases.

### Order-Dependent Test Failure Pattern

```cpp

Test A: writes global_counter = 5     ✓ passes
Test B: reads global_counter, expects 0  ✗ FAILS (sees 5)

Shuffle order:
Test B: reads global_counter = 0      ✓ passes  
Test A: writes global_counter = 5     ✓ passes
                                      → "tests pass sometimes"

```

### Isolation Strategies

| Problem | Solution |
| --- | --- |
| Global variable | Wrap in class, inject as dependency |
| Singleton | Replace with injectable interface |
| File system side effects | Use temp directory per test |
| Static local variable | Reset function or rebuild object |
| Shared container | Create fresh in `SetUp()` |

---

## Self-Assessment

### Q1: Show a test that depends on the execution order of other tests due to shared global state

**Answer:**

```cpp

#include <cassert>
#include <iostream>
#include <string>
#include <vector>

// ═══════════ THE PROBLEM: shared mutable global ═══════════

// Global state — every test shares this
int global_request_count = 0;
std::vector<std::string> global_log;

void handle_request(const std::string& path) {
    global_request_count++;
    global_log.push_back("GET " + path);
}

// ─── Test A: expects clean state ───
void test_first_request() {
    handle_request("/home");
    assert(global_request_count == 1);    // ✓ if run first
    assert(global_log.size() == 1);       // ✗ if run after test B!
    std::cout << "test_first_request passed\n";
}

// ─── Test B: also modifies global ───
void test_multiple_requests() {
    handle_request("/api/v1");
    handle_request("/api/v2");
    assert(global_request_count == 2);    // ✗ FAILS if run after test A!
    // Expected 2, but global_request_count is 3 (1 from A + 2 from B)
    std::cout << "test_multiple_requests passed\n";
}

int main() {
    // Run in order A, B — A passes, B fails
    test_first_request();

    try {
        test_multiple_requests();  // assertion fails: count is 3, not 2
    } catch (...) {
        std::cout << "test_multiple_requests FAILED\n";
        std::cout << "  global_request_count = " << global_request_count << "\n";
        std::cout << "  global_log.size() = " << global_log.size() << "\n";
    }

    return 0;
}
// Output:
// test_first_request passed
// test_multiple_requests FAILED (assertion: 3 != 2)

```

### Q2: Refactor by introducing test fixtures that reset state in SetUp/TearDown

**Answer:**

```cpp

#include <gtest/gtest.h>
#include <string>
#include <vector>
#include <memory>

// ═══════════ SOLUTION 1: Extract state into a class ═══════════

class RequestHandler {
    int request_count_ = 0;
    std::vector<std::string> log_;
public:
    void handle(const std::string& path) {
        request_count_++;
        log_.push_back("GET " + path);
    }
    int count() const { return request_count_; }
    const std::vector<std::string>& log() const { return log_; }
    void reset() { request_count_ = 0; log_.clear(); }
};

// ═══════════ SOLUTION 2: Test fixture with fresh instance per test ═══════════

class RequestHandlerTest : public ::testing::Test {
protected:
    RequestHandler handler;  // Fresh instance for EVERY test

    void SetUp() override {
        // handler is already default-constructed and clean
        // If using a shared resource, reset it here
    }

    void TearDown() override {
        // Optional cleanup (close files, delete temp dirs)
    }
};

TEST_F(RequestHandlerTest, FirstRequest) {
    handler.handle("/home");
    EXPECT_EQ(handler.count(), 1);         // Always 1
    EXPECT_EQ(handler.log().size(), 1u);   // Always 1
}

TEST_F(RequestHandlerTest, MultipleRequests) {
    handler.handle("/api/v1");
    handler.handle("/api/v2");
    EXPECT_EQ(handler.count(), 2);         // Always 2 (fresh handler)
    EXPECT_EQ(handler.log().size(), 2u);
}

TEST_F(RequestHandlerTest, NoRequests) {
    EXPECT_EQ(handler.count(), 0);         // Always 0
    EXPECT_TRUE(handler.log().empty());
}

// ═══════════ SOLUTION 3: For unavoidable globals, reset in SetUp ═══════════

// Sometimes you deal with legacy singletons
class LegacyDatabase {
    static LegacyDatabase* instance_;
    std::vector<std::string> records_;
    LegacyDatabase() = default;
public:
    static LegacyDatabase& instance() {
        if (!instance_) instance_ = new LegacyDatabase();
        return *instance_;
    }
    void insert(const std::string& r) { records_.push_back(r); }
    size_t size() const { return records_.size(); }
    void clear() { records_.clear(); }  // Add reset method
};
LegacyDatabase* LegacyDatabase::instance_ = nullptr;

class LegacyDbTest : public ::testing::Test {
protected:
    void SetUp() override {
        LegacyDatabase::instance().clear();  // Reset before each test!
    }
};

TEST_F(LegacyDbTest, InsertOne) {
    LegacyDatabase::instance().insert("record1");
    EXPECT_EQ(LegacyDatabase::instance().size(), 1u);  // Reliable
}

TEST_F(LegacyDbTest, InsertTwo) {
    LegacyDatabase::instance().insert("a");
    LegacyDatabase::instance().insert("b");
    EXPECT_EQ(LegacyDatabase::instance().size(), 2u);  // Always 2
}

```

### Q3: Explain why test isolation enables parallel test execution with ctest -j

**Answer:**

```cpp

// ═══════════ Why isolation enables ctest -j (parallel) ═══════════

/*
ctest -j8 runs up to 8 test executables simultaneously:

  ┌──────────────────────────────────────────────────────────────┐
  │  Thread/Process 1: test_parser     ─────▶  ✓                │
  │  Thread/Process 2: test_database   ─────▶  ✓                │
  │  Thread/Process 3: test_network    ─────▶  ✓                │
  │  Thread/Process 4: test_auth       ─────▶  ✓                │
  │              (all running at the same time)                  │
  └──────────────────────────────────────────────────────────────┘

  Without isolation:
  ┌──────────────────────────────────────────────────────────────┐
  │  test_parser writes /tmp/output.txt                         │
  │  test_network reads  /tmp/output.txt   ←── RACE CONDITION   │
  │  test_database also writes /tmp/output.txt ← DATA CORRUPT   │
  └──────────────────────────────────────────────────────────────┘
*/

// Practical example: ensuring tests use separate temp directories

#include <gtest/gtest.h>
#include <filesystem>
#include <fstream>
#include <string>

namespace fs = std::filesystem;

class FileProcessorTest : public ::testing::Test {
protected:
    fs::path test_dir_;

    void SetUp() override {
        // Each test gets a unique temp directory
        test_dir_ = fs::temp_directory_path() /
                     ("test_" + std::to_string(
                         std::hash<std::string>{}(
                             ::testing::UnitTest::GetInstance()
                                 ->current_test_info()->name())));
        fs::create_directories(test_dir_);
    }

    void TearDown() override {
        fs::remove_all(test_dir_);  // Clean up
    }

    // Helper: write a file in the test's own directory
    void write_file(const std::string& name, const std::string& content) {
        std::ofstream out(test_dir_ / name);
        out << content;
    }

    std::string read_file_content(const std::string& name) {
        std::ifstream in(test_dir_ / name);
        return std::string(std::istreambuf_iterator<char>(in),
                           std::istreambuf_iterator<char>());
    }
};

TEST_F(FileProcessorTest, WritesOutput) {
    write_file("input.txt", "hello world");
    auto content = read_file_content("input.txt");
    EXPECT_EQ(content, "hello world");
    // Safe: uses its OWN directory, no conflict with parallel tests
}

TEST_F(FileProcessorTest, HandlesEmptyFile) {
    write_file("empty.txt", "");
    EXPECT_TRUE(read_file_content("empty.txt").empty());
}

```

```cmake

# ═══════════ CMakeLists.txt — enable parallel testing ═══════════
enable_testing()

add_executable(test_parser   test_parser.cpp)
add_executable(test_database test_database.cpp)
add_executable(test_network  test_network.cpp)

add_test(NAME parser_tests   COMMAND test_parser)
add_test(NAME database_tests COMMAND test_database)
add_test(NAME network_tests  COMMAND test_network)

# Run: ctest -j8 --output-on-failure
# All 3 test executables run simultaneously

```

**Key principles:**

- **No shared files** — each test uses `SetUp()` to create unique temp paths
- **No shared globals** — extract state into fixture members
- **No assumed ordering** — use `--gtest_shuffle` to prove independence
- **Process-level isolation** — `ctest -j` runs separate executables in parallel (strongest isolation)
- GTest `--gtest_filter` can run subsets; combine with `ctest --parallel` for maximum speed

---

## Notes

- `--gtest_shuffle` randomizes test order — reveals hidden dependencies immediately
- `ctest -j$(nproc)` uses all CPU cores for parallel test execution
- For thread-safety inside a single test binary, GTest supports `--gtest_repeat=N` to stress-test isolation
- If you can't remove a global: at minimum, add a `reset()` method and call it in `SetUp()`
- The gold standard: each test creates everything it needs and cleans up after itself
