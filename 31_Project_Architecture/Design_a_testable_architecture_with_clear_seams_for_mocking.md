# Design a testable architecture with clear seams for mocking

**Category:** Project Architecture

---

## Topic Overview

A **seam** is a point where you can change behavior without editing the code — typically an interface boundary. Testable architecture means designing classes so that external dependencies can be replaced with test doubles. This requires deliberate interface design, dependency injection, and avoiding static/global state.

### Types of Seams in C++

| Seam Type | Mechanism | Overhead | Flexibility |
| --- | --- | --- | --- |
| **Virtual interface** | `virtual` + `override` | vtable dispatch | Runtime swap |
| **Template parameter** | Template injection | None (inlined) | Compile-time swap |
| **Function pointer/std::function** | Callback injection | Indirect call | Runtime swap |
| **Link-time** | Swap `.o` files / weak symbols | None | Build-time swap |
| **Preprocessor** | `#ifdef TESTING` | None | Build-time swap |

---

## Self-Assessment

### Q1: Design seams using virtual interfaces

**Answer:**

```cpp

#include <chrono>
#include <string>
#include <memory>

// === Seam 1: Time provider ===
class IClock {
public:
    virtual ~IClock() = default;
    using TimePoint = std::chrono::system_clock::time_point;
    virtual TimePoint now() const = 0;
};

class SystemClock : public IClock {
public:
    TimePoint now() const override {
        return std::chrono::system_clock::now();
    }
};

class FakeClock : public IClock {
public:
    TimePoint now() const override { return fixed_time_; }
    void set_time(TimePoint t) { fixed_time_ = t; }
    void advance(std::chrono::milliseconds ms) { fixed_time_ += ms; }
private:
    TimePoint fixed_time_ = std::chrono::system_clock::now();
};

// === Seam 2: Random number generator ===
class IRng {
public:
    virtual ~IRng() = default;
    virtual int generate(int min, int max) = 0;
};

class RealRng : public IRng {
public:
    int generate(int min, int max) override {
        std::uniform_int_distribution<int> dist(min, max);
        return dist(engine_);
    }
private:
    std::mt19937 engine_{std::random_device{}()};
};

class DeterministicRng : public IRng {
public:
    explicit DeterministicRng(std::vector<int> values)
        : values_(std::move(values)) {}
    int generate(int, int) override {
        return values_[index_++ % values_.size()];
    }
private:
    std::vector<int> values_;
    size_t index_ = 0;
};

// === Service using seams ===
class SessionManager {
public:
    SessionManager(IClock& clock, IRng& rng)
        : clock_(clock), rng_(rng) {}

    std::string create_session(const std::string& user) {
        auto now = clock_.now();
        auto token = std::to_string(rng_.generate(100000, 999999));
        sessions_[token] = {user, now};
        return token;
    }

    bool is_valid(const std::string& token,
                   std::chrono::minutes max_age) const {
        auto it = sessions_.find(token);
        if (it == sessions_.end()) return false;
        return (clock_.now() - it->second.created) < max_age;
    }

private:
    struct Session {
        std::string user;
        IClock::TimePoint created;
    };
    IClock& clock_;
    IRng& rng_;
    std::unordered_map<std::string, Session> sessions_;
};

```

### Q2: Test with controlled seams

**Answer:**

```cpp

#include <gtest/gtest.h>

TEST(SessionManagerTest, CreateSessionReturnsToken) {
    FakeClock clock;
    DeterministicRng rng({123456});
    SessionManager mgr(clock, rng);

    auto token = mgr.create_session("alice");
    EXPECT_EQ(token, "123456");
}

TEST(SessionManagerTest, SessionExpiresAfterTimeout) {
    FakeClock clock;
    DeterministicRng rng({111111});
    SessionManager mgr(clock, rng);

    auto token = mgr.create_session("bob");
    EXPECT_TRUE(mgr.is_valid(token, std::chrono::minutes(30)));

    // Advance clock past expiry
    clock.advance(std::chrono::minutes(31));
    EXPECT_FALSE(mgr.is_valid(token, std::chrono::minutes(30)));
}

TEST(SessionManagerTest, InvalidTokenIsNotValid) {
    FakeClock clock;
    DeterministicRng rng({999999});
    SessionManager mgr(clock, rng);

    EXPECT_FALSE(mgr.is_valid("nonexistent", std::chrono::minutes(30)));
}

```

### Q3: Template-based seams for zero-overhead testing

**Answer:**

```cpp

// === Compile-time seams via templates ===
template<typename Clock, typename Rng>
class SessionManagerT {
public:
    SessionManagerT(Clock& clock, Rng& rng)
        : clock_(clock), rng_(rng) {}

    std::string create_session(const std::string& user) {
        auto now = clock_.now();
        auto token = std::to_string(rng_.generate(100000, 999999));
        sessions_[token] = {user, now};
        return token;
    }

    // ... same interface ...

private:
    Clock& clock_;
    Rng& rng_;
    // ...
};

// Production: uses real implementations, zero virtual dispatch
// using ProdSessionMgr = SessionManagerT<SystemClock, RealRng>;

// Test: uses fakes, still zero virtual dispatch
// using TestSessionMgr = SessionManagerT<FakeClock, DeterministicRng>;

// === Link-time seam (for C-style APIs like hardware) ===
// In production: link real_timer.o
// In tests: link fake_timer.o

// timer.h:
uint32_t get_tick_count();  // Declaration only

// real_timer.cpp (production):
// uint32_t get_tick_count() { return HAL_GetTick(); }

// fake_timer.cpp (test):
static uint32_t fake_ticks = 0;
uint32_t get_tick_count() { return fake_ticks; }
void set_fake_ticks(uint32_t t) { fake_ticks = t; }

```

---

## Notes

- **Every external dependency should be behind a seam** — time, random, filesystem, network, database
- `FakeClock` is the most common seam — testing time-dependent code without `sleep()`
- Virtual interface seams: most flexible, slight overhead from vtable dispatch
- Template seams: zero overhead, but can't swap at runtime (good for embedded)
- Link-time seams: useful for replacing C APIs (hardware HAL) without modifying caller code
- Avoid `static` and global state — it creates implicit dependencies that can't be replaced
- If a class is hard to test, it's a design problem, not a testing problem
