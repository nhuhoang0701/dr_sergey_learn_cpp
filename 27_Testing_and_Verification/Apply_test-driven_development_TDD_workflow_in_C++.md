# Apply test-driven development (TDD) workflow in C++

**Category:** Testing & Verification  
**Item:** #680  
**Reference:** <https://github.com/catchorg/Catch2>  

---

## Topic Overview

**Test-Driven Development (TDD)** is a workflow where you write a **failing test first**, then write the **minimal code** to make it pass, then **refactor**. This "Red-Green-Refactor" cycle produces code that is testable by design, with complete test coverage from the start.

### The Red-Green-Refactor Cycle

```cpp

    ┌──── RED ─────┐
    │ Write a test  │
    │ that FAILS    │
    └──────┬────────┘
           ▼
    ┌──── GREEN ───┐
    │ Write MINIMAL│
    │ code to pass │
    └──────┬───────┘
           ▼
    ┌── REFACTOR ──┐
    │ Clean up code│
    │ tests still  │
    │ pass         │
    └──────┬───────┘
           │
           └──→ repeat

```

### Catch2 Quick Reference

| Macro                         | Purpose                                 |
| --- | --- |
| `TEST_CASE("name")`          | Define a test case                      |
| `SECTION("name")`            | Subsection within a test case           |
| `REQUIRE(expr)`              | Assert (test fails if false)            |
| `CHECK(expr)`                | Assert (reports failure, continues)     |
| `REQUIRE_THROWS_AS(expr, T)` | Assert exception type thrown            |

---

## Self-Assessment

### Q1: Write a failing test with Catch2 REQUIRE before implementing the function it tests

**Answer:**

```cpp

// ═══════════ Step 1: RED — Write the test FIRST ═══════════
// File: test_calculator.cpp
#include <catch2/catch_test_macros.hpp>

// Forward declaration — function doesn't exist yet!
int add(int a, int b);

TEST_CASE("add() computes the sum of two integers") {
    REQUIRE(add(2, 3) == 5);
    REQUIRE(add(-1, 1) == 0);
    REQUIRE(add(0, 0) == 0);
    REQUIRE(add(-5, -3) == -8);
    REQUIRE(add(INT_MAX, 0) == INT_MAX);
}
// COMPILE ERROR: add() is not defined → this is the "RED" state

// ═══════════ Step 2: GREEN — Minimal implementation ═══════════
// File: calculator.hpp
int add(int a, int b) {
    return a + b;  // Simplest code that makes all tests pass
}
// TESTS PASS → this is the "GREEN" state

// ═══════════ Step 3: No refactor needed (already clean) ═══════════
// In more complex cases, you'd clean up duplicated code here while
// ensuring tests still pass.

```

### Q2: Follow the red-green-refactor cycle: failing test, minimal implementation, clean refactor

**Answer:**

```cpp

// Full TDD cycle for a Stack<T> class

// ═══════════ Iteration 1: push and size ═══════════

// RED: Write failing test
#include <catch2/catch_test_macros.hpp>
#include <stdexcept>

// Forward declare
template<typename T> class Stack;

TEST_CASE("Stack push and size") {
    Stack<int> s;
    REQUIRE(s.size() == 0);
    REQUIRE(s.empty());

    s.push(42);
    REQUIRE(s.size() == 1);
    REQUIRE(!s.empty());
}
// → FAILS: Stack doesn't exist

// GREEN: Minimal implementation
#include <vector>

template<typename T>
class Stack {
    std::vector<T> data_;
public:
    void push(const T& val) { data_.push_back(val); }
    size_t size() const { return data_.size(); }
    bool empty() const { return data_.empty(); }
};
// → PASSES

// ═══════════ Iteration 2: pop and top ═══════════

// RED: Add new tests (existing tests still run)
TEST_CASE("Stack pop and top") {
    Stack<int> s;
    s.push(10);
    s.push(20);

    REQUIRE(s.top() == 20);

    s.pop();
    REQUIRE(s.top() == 10);
    REQUIRE(s.size() == 1);
}

TEST_CASE("Stack pop on empty throws") {
    Stack<int> s;
    REQUIRE_THROWS_AS(s.pop(), std::underflow_error);
    REQUIRE_THROWS_AS(s.top(), std::underflow_error);
}
// → FAILS: top() and pop() don't exist

// GREEN: Add methods
// (add to Stack class):
//   const T& top() const {
//       if (empty()) throw std::underflow_error("top on empty stack");
//       return data_.back();
//   }
//   void pop() {
//       if (empty()) throw std::underflow_error("pop on empty stack");
//       data_.pop_back();
//   }
// → PASSES

// ═══════════ Iteration 3: REFACTOR ═══════════
// Notice: the emptiness check is duplicated in top() and pop().
// Extract a helper:
//   void check_not_empty() const {
//       if (empty()) throw std::underflow_error("stack is empty");
//   }
// Then top() and pop() call check_not_empty().
// Run tests → still PASS → refactor is safe.

```

### Q3: Show how TDD drives better API design: the test forces a clean, testable interface

**Answer:**

```cpp

// ═══════════ Without TDD: untestable design ═══════════
// Developer writes implementation first, tests as afterthought

class BadUserService {
public:
    bool register_user(const std::string& name) {
        // Directly connects to database — untestable without real DB!
        auto conn = DatabasePool::get_connection();  // Hard dependency
        conn.execute("INSERT INTO users ...");
        EmailService::send_welcome(name);            // Hard dependency
        Logger::log("User registered");              // Hard dependency
        return true;
    }
};
// Testing this requires: running DB, email server, log infrastructure
// → Slow, flaky, rarely tested

// ═══════════ With TDD: testable design emerges naturally ═══════════
// Writing the TEST FIRST forces you to design injectable interfaces

// Step 1: Write the test — you're "designing the API from the consumer side"
#include <catch2/catch_test_macros.hpp>
#include <string>
#include <vector>
#include <functional>

struct UserRepository {
    virtual ~UserRepository() = default;
    virtual bool save(const std::string& name) = 0;
};

struct Notifier {
    virtual ~Notifier() = default;
    virtual void notify(const std::string& msg) = 0;
};

class UserService {
    UserRepository& repo_;
    Notifier& notifier_;
public:
    UserService(UserRepository& r, Notifier& n) : repo_(r), notifier_(n) {}

    bool register_user(const std::string& name) {
        if (name.empty()) return false;
        if (!repo_.save(name)) return false;
        notifier_.notify("Welcome " + name);
        return true;
    }
};

// Step 2: Test with fakes — no real DB or email needed
struct FakeRepo : UserRepository {
    std::vector<std::string> saved;
    bool should_fail = false;
    bool save(const std::string& name) override {
        if (should_fail) return false;
        saved.push_back(name);
        return true;
    }
};

struct FakeNotifier : Notifier {
    std::vector<std::string> messages;
    void notify(const std::string& msg) override {
        messages.push_back(msg);
    }
};

TEST_CASE("UserService registers user and notifies") {
    FakeRepo repo;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE(svc.register_user("Alice"));
    REQUIRE(repo.saved.size() == 1);
    REQUIRE(repo.saved[0] == "Alice");
    REQUIRE(notifier.messages[0] == "Welcome Alice");
}

TEST_CASE("UserService rejects empty name") {
    FakeRepo repo;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE_FALSE(svc.register_user(""));
    REQUIRE(repo.saved.empty());
}

TEST_CASE("UserService handles DB failure") {
    FakeRepo repo;
    repo.should_fail = true;
    FakeNotifier notifier;
    UserService svc(repo, notifier);

    REQUIRE_FALSE(svc.register_user("Bob"));
    REQUIRE(notifier.messages.empty());  // No notification if save failed
}

// TDD forced us to:
// 1. Define clean interfaces (UserRepository, Notifier)
// 2. Use dependency injection (constructor params)
// 3. Separate concerns (save vs notify vs validation)
// 4. Handle edge cases (empty name, DB failure)
// → The TEST drove the DESIGN

```

---

## Notes

- Build system: `find_package(Catch2 3 REQUIRED)` + `target_link_libraries(tests Catch2::Catch2WithMain)`
- Run single tests: `./tests "Stack push and size"` (Catch2 supports name-based filtering)
- TDD works best for **pure logic** (calculations, data transformations, state machines)
- For I/O-bound code, TDD naturally pushes toward dependency injection — this is a feature, not overhead
- Combine with sanitizers: `cmake -DCMAKE_CXX_FLAGS="-fsanitize=address,undefined"` catches memory bugs TDD alone misses
